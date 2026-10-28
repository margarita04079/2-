using Microsoft.EntityFrameworkCore;
using VisitorApi.Data;
using VisitorApi.Services;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddScoped<IVisitorService, VisitorService>();

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.MapControllers();
app.Run();

namespace VisitorApi.Data
{
    using Microsoft.EntityFrameworkCore;
    using VisitorApi.Models;

    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
        public DbSet<Visitor> Visitors => Set<Visitor>();
        public DbSet<VisitorAction> VisitorActions => Set<VisitorAction>();

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder.Entity<Visitor>(entity =>
            {
                entity.HasKey(v => v.Id);
                entity.Property(v => v.Name).IsRequired().HasMaxLength(200);
                entity.Property(v => v.TrustLevel).IsRequired();
                entity.HasMany(v => v.Actions)
                      .WithOne(a => a.Visitor)
                      .HasForeignKey(a => a.VisitorId)
                      .OnDelete(DeleteBehavior.Cascade);
            });

            modelBuilder.Entity<VisitorAction>(entity =>
            {
                entity.HasKey(a => a.Id);
                entity.Property(a => a.ActionType).IsRequired().HasMaxLength(50);
                entity.Property(a => a.TrustChange).IsRequired();
                entity.Property(a => a.PerformedAt).IsRequired();
                entity.HasIndex(a => new { a.VisitorId, a.ActionType }).IsUnique();
            });
        }
    }
}

namespace VisitorApi.Models
{
    public class Visitor
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public int TrustLevel { get; set; } = 0;
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
        public List<VisitorAction> Actions { get; set; } = new();
    }

    public class VisitorAction
    {
        public int Id { get; set; }
        public int VisitorId { get; set; }
        public Visitor Visitor { get; set; } = null!;
        public string ActionType { get; set; } = string.Empty;
        public int TrustChange { get; set; }
        public DateTime PerformedAt { get; set; } = DateTime.UtcNow;
    }

    public enum ActionType
    {
        Introduce,
        OfferHelp,
        Threaten
    }

    public static class ActionTypeExtensions
    {
        public static int GetTrustChange(this ActionType action) => action switch
        {
            ActionType.Introduce => 20,
            ActionType.OfferHelp => 30,
            ActionType.Threaten => -20,
            _ => 0
        };

        public static bool TryParseActionType(string value, out ActionType action)
        {
            return Enum.TryParse(value, ignoreCase: true, out action)
                   && Enum.IsDefined(typeof(ActionType), action);
        }
    }
}

namespace VisitorApi.Dtos
{
    public class CreateVisitorDto
    {
        public string Name { get; set; } = string.Empty;
    }

    public class PerformActionDto
    {
        public string Action { get; set; } = string.Empty;
    }

    public class VisitorResponseDto
    {
        public int Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public int TrustLevel { get; set; }
        public DateTime CreatedAt { get; set; }
        public List<ActionHistoryDto> Actions { get; set; } = new();
    }

    public class ActionHistoryDto
    {
        public string Action { get; set; } = string.Empty;
        public int TrustChange { get; set; }
        public DateTime PerformedAt { get; set; }
    }

    public class ActionResultDto
    {
        public int VisitorId { get; set; }
        public string VisitorName { get; set; } = string.Empty;
        public string Action { get; set; } = string.Empty;
        public int TrustChange { get; set; }
        public int CurrentTrustLevel { get; set; }
        public DateTime PerformedAt { get; set; }
    }

    public class AccessResultDto
    {
        public int VisitorId { get; set; }
        public int TrustLevel { get; set; }
        public bool HasAccess { get; set; }
    }
}

namespace VisitorApi.Services
{
    using Microsoft.EntityFrameworkCore;
    using VisitorApi.Data;
    using VisitorApi.Dtos;
    using VisitorApi.Models;

    public interface IVisitorService
    {
        Task<VisitorResponseDto> CreateVisitorAsync(CreateVisitorDto dto);
        Task<List<VisitorResponseDto>> GetAllVisitorsAsync();
        Task<VisitorResponseDto?> GetVisitorByIdAsync(int id);
        Task<(ActionResultDto? Result, string? Error)> PerformActionAsync(int id, PerformActionDto dto);
        Task<AccessResultDto?> CheckAccessAsync(int id);
    }

    public class VisitorService : IVisitorService
    {
        private readonly AppDbContext _db;

        public VisitorService(AppDbContext db)
        {
            _db = db;
        }

        public async Task<VisitorResponseDto> CreateVisitorAsync(CreateVisitorDto dto)
        {
            var visitor = new Visitor
            {
                Name = dto.Name.Trim(),
                TrustLevel = 0,
                CreatedAt = DateTime.UtcNow
            };
            _db.Visitors.Add(visitor);
            await _db.SaveChangesAsync();
            return MapToDto(visitor);
        }

        public async Task<List<VisitorResponseDto>> GetAllVisitorsAsync()
        {
            var visitors = await _db.Visitors
                .Include(v => v.Actions)
                .OrderBy(v => v.Id)
                .ToListAsync();
            return visitors.Select(MapToDto).ToList();
        }

        public async Task<VisitorResponseDto?> GetVisitorByIdAsync(int id)
        {
            var visitor = await _db.Visitors
                .Include(v => v.Actions)
                .FirstOrDefaultAsync(v => v.Id == id);
            return visitor is null ? null : MapToDto(visitor);
        }

        public async Task<(ActionResultDto? Result, string? Error)> PerformActionAsync(int id, PerformActionDto dto)
        {
            if (string.IsNullOrWhiteSpace(dto.Action) ||
                !ActionTypeExtensions.TryParseActionType(dto.Action, out var actionType))
            {
                return (null, "Invalid action. Allowed values: Introduce, OfferHelp, Threaten.");
            }

            var visitor = await _db.Visitors
                .Include(v => v.Actions)
                .FirstOrDefaultAsync(v => v.Id == id);

            if (visitor is null)
                return (null, "Visitor not found.");

            var actionName = actionType.ToString();

            if (visitor.Actions.Any(a => a.ActionType == actionName))
                return (null, "Action already performed by this visitor.");

            var trustChange = actionType.GetTrustChange();
            var newTrust = Math.Clamp(visitor.TrustLevel + trustChange, 0, 100);
            var actualChange = newTrust - visitor.TrustLevel;

            visitor.TrustLevel = newTrust;

            var action = new VisitorAction
            {
                VisitorId = visitor.Id,
                ActionType = actionName,
                TrustChange = actualChange,
                PerformedAt = DateTime.UtcNow
            };
            _db.VisitorActions.Add(action);

            try
            {
                await _db.SaveChangesAsync();
            }
            catch (DbUpdateException)
            {
                return (null, "Action already performed by this visitor.");
            }

            return (new ActionResultDto
            {
                VisitorId = visitor.Id,
                VisitorName = visitor.Name,
                Action = actionName,
                TrustChange = actualChange,
                CurrentTrustLevel = visitor.TrustLevel,
                PerformedAt = action.PerformedAt
            }, null);
        }

        public async Task<AccessResultDto?> CheckAccessAsync(int id)
        {
            var visitor = await _db.Visitors.FirstOrDefaultAsync(v => v.Id == id);
            if (visitor is null) return null;

            return new AccessResultDto
            {
                VisitorId = visitor.Id,
                TrustLevel = visitor.TrustLevel,
                HasAccess = visitor.TrustLevel >= 50
            };
        }

        private static VisitorResponseDto MapToDto(Visitor v) => new()
        {
            Id = v.Id,
            Name = v.Name,
            TrustLevel = v.TrustLevel,
            CreatedAt = v.CreatedAt,
            Actions = v.Actions
                .OrderBy(a => a.PerformedAt)
                .Select(a => new ActionHistoryDto
                {
                    Action = a.ActionType,
                    TrustChange = a.TrustChange,
                    PerformedAt = a.PerformedAt
                }).ToList()
        };
    }
}

namespace VisitorApi.Controllers
{
    using Microsoft.AspNetCore.Mvc;
    using VisitorApi.Dtos;
    using VisitorApi.Services;

    [ApiController]
    [Route("api/visitors")]
    public class VisitorsController : ControllerBase
    {
        private readonly IVisitorService _service;

        public VisitorsController(IVisitorService service)
        {
            _service = service;
        }

        [HttpPost]
        [ProducesResponseType(typeof(VisitorResponseDto), StatusCodes.Status201Created)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<IActionResult> Create([FromBody] CreateVisitorDto dto)
        {
            if (dto is null || string.IsNullOrWhiteSpace(dto.Name))
                return BadRequest(new { error = "Name is required." });

            var result = await _service.CreateVisitorAsync(dto);
            return CreatedAtAction(nameof(GetById), new { id = result.Id }, result);
        }

        [HttpGet]
        [ProducesResponseType(typeof(List<VisitorResponseDto>), StatusCodes.Status200OK)]
        public async Task<IActionResult> GetAll()
        {
            var result = await _service.GetAllVisitorsAsync();
            return Ok(result);
        }

        [HttpGet("{id:int}")]
        [ProducesResponseType(typeof(VisitorResponseDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> GetById(int id)
        {
            var result = await _service.GetVisitorByIdAsync(id);
            return result is null
                ? NotFound(new { error = "Visitor not found." })
                : Ok(result);
        }

        [HttpPost("{id:int}/actions")]
        [ProducesResponseType(typeof(ActionResultDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> PerformAction(int id, [FromBody] PerformActionDto dto)
        {
            if (dto is null)
                return BadRequest(new { error = "Request body is required." });

            var (result, error) = await _service.PerformActionAsync(id, dto);

            if (error == "Visitor not found.")
                return NotFound(new { error });

            if (error is not null)
                return BadRequest(new { error });

            return Ok(result);
        }

        [HttpGet("{id:int}/access")]
        [ProducesResponseType(typeof(AccessResultDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> CheckAccess(int id)
        {
            var result = await _service.CheckAccessAsync(id);
            return result is null
                ? NotFound(new { error = "Visitor not found." })
                : Ok(result);
        }
    }
}
