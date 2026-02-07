# MPC-Plus Comprehensive Code Review

**Reviewer:** Claude (Automated Code Review)
**Date:** 2026-02-07
**Codebase Snapshot:** Current `main` branch
**Scope:** Full-stack review of C# API, Python data pipeline, and supporting infrastructure

---

## 1. Executive Summary

MPC-Plus is a well-architected radiation therapy machine QA monitoring system that reads
MPC data from Varian TrueBeam linacs, stores it in Supabase (PostgreSQL), and exposes it
via a REST API for a dashboard frontend. The project demonstrates solid software
engineering fundamentals: proper separation of concerns via the repository pattern,
dependency injection, interface-based design, and reasonable test coverage.

### What the team did well

- **Clean architecture:** The C# API follows the repository pattern with interface
  abstractions, entity-model separation, and DI throughout. This is genuinely
  well-organized for a student project.
- **Sensible physics delegation:** Rather than re-implementing beam physics calculations,
  the team correctly delegates flatness/symmetry analysis to `pylinac`
  (a peer-reviewed medical physics library) with appropriate protocol settings.
- **Secrets management:** No hardcoded credentials. Environment variables loaded via
  `.env` with proper `.gitignore` entries.
- **Test infrastructure:** 51 xUnit tests covering both repository logic and controller
  behavior, using proper mocking (Moq) and fluent assertions.
- **Graceful degradation:** The API falls back to in-memory repositories when Supabase
  credentials are missing, enabling local development without a database.

### Top 5 priorities for the remaining month

1. **CRITICAL: Implement authentication and authorization on the approval endpoints.**
   Anyone can currently approve beam checks by sending a POST with an arbitrary name.
   This is the single most important fix for clinical use.

2. **CRITICAL: Implement the `DetermineCheckStatus` / `DetermineGeoCheckStatus` methods
   in `ResultsController`.** They currently hardcode `"pass"` for every check, making
   the monthly calendar view useless for catching failures.

3. **HIGH: Fix the DOC factor division-by-zero risk** and add validation that `MpcRel`
   is never zero before computing `MsdAbs / MpcRel`.

4. **HIGH: Document all threshold values and their clinical sources.** The report service
   hardcodes thresholds (2%, 3%, 2mm, etc.) that differ from the configurable threshold
   system used in `BeamsController`. These need to be reconciled and sourced to a
   standard.

5. **MEDIUM: Fix the broken `folder_monitor.py`** which has a syntax error in the
   `FolderMonitor.__init__` method that will crash at runtime.

---

## 2. Critical Issues (Must fix before any clinical use)

### C-1. No authentication or authorization on API endpoints

**Files:** `src/api/Program.cs`, all Controllers
**Severity:** CRITICAL
**Category:** Security / Clinical Safety

The entire API is unauthenticated. Any network client can:
- Approve or un-approve beam checks (`POST /api/beams/accept`)
- Approve geometry checks (`POST /api/geochecks/accept`)
- Delete measurements (`DELETE /api/beams/{id}`)
- Modify thresholds (`POST /api/thresholds`)
- Create, update, or delete DOC factors

The `ApprovedBy` field is a free-form string supplied by the caller with no verification:

```csharp
// src/api/Controllers/BeamsController.cs:218-219
beam.ApprovedBy = request.ApprovedBy;  // Any string accepted, no auth check
beam.ApprovedDate = DateTime.UtcNow;
```

A `LoginRequest` model exists but has no corresponding endpoint or middleware.

**Why this matters clinically:** In radiation therapy QA, the approval workflow exists
because regulatory bodies (state radiation control programs, The Joint Commission, NRC)
require that a qualified medical physicist review and sign off on machine performance
data. An unauthenticated approval system cannot demonstrate that the person who approved
a measurement was actually authorized to do so. This could jeopardize the clinic's
accreditation and, more importantly, patient safety.

**Recommendation:** Implement Supabase Auth (which is already available in the Supabase
SDK you're using). At minimum:

1. Add JWT authentication middleware to the API
2. Protect the `/accept` endpoints with `[Authorize(Roles = "physicist")]`
3. Derive `ApprovedBy` from the authenticated user's identity, not from the request body
4. Add an audit log table recording who approved what and when

```csharp
// Example: Derive approver from JWT claims, not request body
[Authorize(Roles = "physicist")]
[HttpPost("accept")]
public async Task<ActionResult> Accept(
    [FromBody] AcceptBeamRequest request,
    CancellationToken cancellationToken)
{
    var approverName = User.FindFirst(ClaimTypes.Name)?.Value
        ?? throw new UnauthorizedAccessException();
    // ... use approverName instead of request.ApprovedBy
}
```

### C-2. ResultsController always returns "pass" for every check

**File:** `src/api/Controllers/ResultsController.cs:157-172`
**Severity:** CRITICAL
**Category:** Clinical Correctness

The monthly calendar view — the primary way physicists would see at-a-glance whether
the machine is performing within tolerance — always shows "pass":

```csharp
private static string DetermineCheckStatus(Beam beam)
{
    // TODO: Implement actual pass/warning/fail logic based on beam metrics
    // For now, return "pass" as default
    return "pass";
}

private static string DetermineGeoCheckStatus(GeoCheck geoCheck)
{
    // TODO: Implement actual pass/warning/fail logic based on geometry check metrics
    // For now, return "pass" as default
    return "pass";
}
```

**Why this matters clinically:** A system that always reports "pass" is worse than no
system at all because it creates false confidence. A physicist relying on this dashboard
could miss a genuine machine malfunction.

**Recommendation:** Reuse the threshold-checking logic that already exists in
`BeamsController.CalculateStatus()`. Refactor it into a shared service:

```csharp
// Create: src/api/Services/StatusCalculationService.cs
public class StatusCalculationService
{
    private readonly IThresholdRepository _thresholdRepository;

    public async Task<string> DetermineBeamStatus(Beam beam, CancellationToken ct)
    {
        var thresholds = await _thresholdRepository.GetAllAsync(ct);
        // Reuse the same logic from BeamsController.CalculateStatus
        // Return "pass", "warning", or "fail"
    }
}
```

### C-3. Division by zero in DOC factor calculation

**File:** `src/api/Repositories/Supabase/SupabaseDocFactorRepository.cs:131`
**Severity:** CRITICAL
**Category:** Clinical Correctness / Data Integrity

```csharp
docFactor.DocFactorValue = docFactor.MsdAbs / docFactor.MpcRel;
```

If `MpcRel` is zero (e.g., a machine malfunction, sensor error, or data entry mistake),
this will produce `Infinity` or `NaN`, which will then be stored in the database and
used to compute absolute dose values. A corrupted DOC factor could silently produce
nonsensical absolute dose readings.

**Recommendation:**

```csharp
if (docFactor.MpcRel == 0)
{
    throw new ArgumentException(
        "MPC relative output (MpcRel) cannot be zero. " +
        "This may indicate a measurement error.");
}
docFactor.DocFactorValue = docFactor.MsdAbs / docFactor.MpcRel;
```

Also validate in the controller before it reaches the repository:

```csharp
[HttpPost]
public async Task<ActionResult<DocFactor>> Create(
    [FromBody] DocFactor docFactor, CancellationToken cancellationToken)
{
    if (docFactor.MpcRel == 0)
        return BadRequest("MPC relative output cannot be zero.");
    if (docFactor.MsdAbs <= 0)
        return BadRequest("Measured absolute output must be positive.");
    // ...
}
```

### C-4. Invalid parse values silently stored as -1

**File:** `src/data_manipulation/ETL/data_extractor.py:97-100`
**Severity:** CRITICAL
**Category:** Clinical Correctness / Data Integrity

When a CSV value can't be parsed as a `Decimal`, the code silently substitutes `-1`:

```python
try:
    dec_val = Decimal(value)
except (ValueError, TypeError, decimal.InvalidOperation):
    dec_val = Decimal(-1)
```

This pattern is repeated in all three extraction methods (eModel, xModel, geoModel).
A value of `-1` for beam output or center shift is clinically plausible (it's within
normal measurement ranges), so downstream systems have no way to distinguish "the parser
failed" from "the machine measured -1%". This could mask data corruption.

**Recommendation:** Use `None` (null) for unparseable values and log a warning:

```python
try:
    dec_val = Decimal(value)
except (ValueError, TypeError, decimal.InvalidOperation):
    logger.warning(f"Could not parse value for '{name}': '{value}'. Setting to None.")
    dec_val = None
```

Then update setters to accept `None` and let the database column be nullable.

### C-5. Hardcoded thresholds inconsistent across codebase

**Files:**
- `src/api/Services/ReportService.cs:205-223` (hardcoded: 2%, 3%, 2mm)
- `src/api/Controllers/BeamsController.cs:243-298` (dynamic from threshold table)
- `src/api/Controllers/ResultsController.cs:157-172` (hardcoded "pass")

**Severity:** CRITICAL
**Category:** Clinical Correctness

Three different parts of the system determine pass/fail status using three different
approaches:

| Component | Approach | Output ±% | Uniformity ±% | Center Shift mm |
|---|---|---|---|---|
| `BeamsController` | Dynamic from `thresholds` table | Configurable | Configurable | Configurable |
| `ReportService` | Hardcoded | 2.0 | 3.0 | 2.0 |
| `ResultsController` | None (always "pass") | N/A | N/A | N/A |

A physicist could configure thresholds to ±1.5% for output in the dashboard but then
generate a PDF report that uses ±2.0% and see different pass/fail results. This is
confusing and potentially dangerous.

**Recommendation:** Create a single `ThresholdService` that all three components use.
The report service should query the same threshold table as the dashboard:

```csharp
public class ThresholdService
{
    private readonly IThresholdRepository _repository;

    // Default thresholds used when no custom thresholds are configured
    private static readonly Dictionary<string, double> Defaults = new()
    {
        ["Relative Output"] = 2.0,
        ["Relative Uniformity"] = 3.0,
        ["Center Shift"] = 2.0,
    };

    public async Task<double> GetThreshold(
        string machineId, string beamType, string metricType, CancellationToken ct)
    {
        var custom = await _repository.FindAsync(machineId, "beam", beamType, metricType, ct);
        return custom?.Value ?? Defaults.GetValueOrDefault(metricType, double.MaxValue);
    }
}
```

---

## 3. Important Issues (Should fix, ordered by impact)

### I-1. Syntax error in folder_monitor.py will crash at runtime

**File:** `src/data_manipulation/file_monitoring/folder_monitor.py:153-176`
**Severity:** HIGH
**Category:** Reliability

The `FolderMonitor.__init__` method has leftover code from a refactor that creates
a syntax error:

```python
class FolderMonitor:
    def __init__(self, idrive_path="iDrive", supabase_url=None, supabase_key=None):
        if isinstance(idrive_path, list):
            self.idrive_paths = [os.path.abspath(p) for p in idrive_path]
        else:
            self.idrive_paths = [os.path.abspath(idrive_path)]

        self.observers = []
        self.handler = iDriveFolderHandler()
            idrive_path (str): Path to the iDrive folder to monitor  # <-- SYNTAX ERROR
            supabase_url (str, optional): ...                        # <-- orphaned docstring

#         self.idrive_path = os.path.abspath(idrive_path)            # <-- commented-out old code
#         self.observer = Observer()
#         self.handler = iDriveFolderHandler(supabase_url, supabase_key)
        self.is_running = False
```

Lines 169-175 are remnants of a docstring and old constructor body that were
incompletely removed during a refactor. Python will raise a `SyntaxError` before the
monitor can start. Additionally, `iDriveFolderHandler()` is constructed without the
`supabase_url` and `supabase_key` arguments that the constructor now expects.

**Recommendation:** Clean up the constructor:

```python
def __init__(self, idrive_path="iDrive", supabase_url=None, supabase_key=None):
    if isinstance(idrive_path, list):
        self.idrive_paths = [os.path.abspath(p) for p in idrive_path]
    else:
        self.idrive_paths = [os.path.abspath(idrive_path)]

    self.observers = []
    self.handler = iDriveFolderHandler(supabase_url, supabase_key)
    self.is_running = False
```

### I-2. Geo model upload has unreachable dead code that will execute incorrectly

**File:** `src/data_manipulation/ETL/Uploader.py:932-1101`
**Severity:** HIGH
**Category:** Reliability

The `geoModelUpload` method has a confusing structure: lines 996-1097 reference
`result_id` (which is never defined) and call upload methods for geometry-specific
tables. These lines are described in comments as "COMMENTED OUT" but they are
**not actually commented out** — they are indented inside the `else` block and will
execute after the beam data upload. This will crash with a `NameError` on `result_id`.

```python
# Line 988-989: uploads basic beam data
result = self.db_adapter.upload_beam_data('beams', data)

# Lines 996-1091: References undefined variable 'result_id'
isocenter_data = {
    'beam_id': result_id,  # <-- NameError: 'result_id' is not defined
    ...
}
```

**Recommendation:** Either properly comment out lines 996-1091 (add `#` to each line),
or complete the implementation by capturing the returned ID:

```python
result_id = self.db_adapter.upload_beam_data('beams', data)
if not result_id:
    logger.error("Failed to upload basic beam data for geo model")
    return False
```

### I-3. Multiple Supabase clients created per request

**File:** `src/api/Extensions/ServiceCollectionExtensions.cs`
**Severity:** HIGH
**Category:** Performance / Resource Management

Each repository registration creates its own `Client` instance inside a `Scoped`
factory. With 6 repositories, each HTTP request creates **6 separate Supabase client
connections**, each calling `InitializeAsync().GetAwaiter().GetResult()` (a synchronous
blocking call on an async method):

```csharp
services.AddScoped<IBeamRepository>(provider =>
{
    // ... This runs for EVERY request, for EVERY repository
    var client = new Client(settings.Url, settings.Key, options);
    client.InitializeAsync().GetAwaiter().GetResult();  // Blocks the thread pool
    return new SupabaseBeamRepository(client, logger);
});
```

**Recommendation:** Register a single `Client` as a singleton and share it:

```csharp
services.AddSingleton<Client>(provider =>
{
    var settings = provider.GetRequiredService<IOptions<SupabaseSettings>>().Value;
    var client = new Client(settings.Url, settings.Key, new SupabaseOptions
    {
        AutoConnectRealtime = false
    });
    client.InitializeAsync().GetAwaiter().GetResult();
    return client;
});

services.AddScoped<IBeamRepository, SupabaseBeamRepository>();
services.AddScoped<IMachineRepository, SupabaseMachineRepository>();
// etc.
```

### I-4. `Console.WriteLine` used for logging in production code

**Files:** `src/api/Program.cs:27-28`, `src/api/Services/ReportService.cs` (15+ instances)
**Severity:** MEDIUM
**Category:** Reliability / Observability

The report service uses `Console.WriteLine` extensively for debug logging:

```csharp
Console.WriteLine($"[DEBUG] SUPABASE_URL: {supabaseUrl}");  // Program.cs:27
Console.WriteLine($"[ReportService] Fetched {beams.Count} beams...");  // ReportService
```

This bypasses the structured logging framework (ILogger) that's properly used elsewhere.
Console output isn't captured in log aggregation systems, can't be filtered by level,
and the `[DEBUG]` line in `Program.cs` prints the full Supabase URL to stdout on every
startup.

**Recommendation:** Replace all `Console.WriteLine` with `ILogger`:

```csharp
_logger.LogDebug("Fetched {BeamCount} beams for range {Start} to {End}",
    beams.Count, request.StartDate, searchEndDate);
```

### I-5. No idempotency protection on data uploads

**File:** `src/data_manipulation/ETL/Uploader.py:176-215`
**Severity:** MEDIUM
**Category:** Data Integrity

If the folder monitor processes the same folder twice (e.g., due to a retry after
partial failure, or the monitor restarting), the `upload_beam_data` method performs an
`INSERT` without checking for duplicates:

```python
response = self.client.table(table_name).insert(serialized_data).execute()
```

The `processed_folders` set in `folder_monitor.py` provides some protection, but it's
only in-memory and lost on restart. There's no database-level deduplication.

**Recommendation:** Use an upsert or check for existing records before inserting:

```python
# Option A: Use upsert (if Supabase supports it for your table)
response = self.client.table(table_name).upsert(
    serialized_data,
    on_conflict="machine_id,type,timestamp"
).execute()

# Option B: Check before insert
existing = self.client.table(table_name).select("id").eq(
    "machine_id", data["machine_id"]
).eq("timestamp", data["timestamp"]).execute()

if existing.data:
    logger.info(f"Record already exists, skipping upload")
    return True
```

### I-6. No retry/timeout on Supabase operations

**Files:** All Supabase repository files
**Severity:** MEDIUM
**Category:** Reliability

If the Supabase service is temporarily unavailable (network blip, maintenance window),
every API call will fail immediately with an unhandled exception. There's no retry logic,
circuit breaker, or timeout configuration.

**Recommendation:** For the remaining month, a simple retry with Polly is sufficient:

```csharp
// In Program.cs or ServiceCollectionExtensions
services.AddScoped<IBeamRepository>(provider =>
{
    var inner = new SupabaseBeamRepository(client, logger);
    return new RetryingBeamRepository(inner, maxRetries: 3);
});
```

### I-7. Beam type detection relies on path substring matching

**File:** `src/data_manipulation/ETL/DataProcessor.py:115-132`
**Severity:** MEDIUM
**Category:** Reliability

Beam type is detected by checking if a key string appears anywhere in the file path:

```python
for key, (model_class, beam_type) in beam_map.items():
    if key in self.data_path:
```

This means a path containing `16e` also matches `6e` (since `"6e" in "16e"` is `True`).
The iteration order happens to check `6e` before `16e` in the dictionary, but Python
dict ordering, while insertion-ordered since 3.7, makes this fragile. A path like
`...BeamCheckTemplate16e/Results.csv` would incorrectly match as `6e`.

**Recommendation:** Check in specificity order (longest match first) or use more precise
matching:

```python
beam_map = OrderedDict([
    ("16e",  (EBeamModel, "16e")),   # Check 16e before 6e
    ("12e",  (EBeamModel, "12e")),
    ("9e",   (EBeamModel, "9e")),
    ("6e",   (EBeamModel, "6e")),
    ("15x",  (XBeamModel, "15x")),   # Check 15x before 10x
    ("2.5x", (XBeamModel, "2.5x")),
    ("10x",  (XBeamModel, "10x")),
    ("6x",   (Geo6xfffModel, "6x")),
])
```

Or better, use regex to match the template name exactly:

```python
import re
match = re.search(r'BeamCheckTemplate(\d+\.?\d*[ex])', self.data_path, re.IGNORECASE)
```

### I-8. CORS configured only for localhost

**File:** `src/api/Program.cs:50-58`
**Severity:** MEDIUM
**Category:** Production Readiness

```csharp
policy.WithOrigins("http://localhost:3000")
```

This will break as soon as the frontend is deployed to any non-localhost URL.

**Recommendation:** Make the allowed origin configurable:

```csharp
var allowedOrigins = builder.Configuration
    .GetSection("Cors:AllowedOrigins")
    .Get<string[]>() ?? new[] { "http://localhost:3000" };

policy.WithOrigins(allowedOrigins)
```

### I-9. Matplotlib figure memory leak in image processing

**File:** `src/data_manipulation/ETL/image_extractor.py:127-161`
**Severity:** MEDIUM
**Category:** Reliability

The comment on lines 146-147 acknowledges the issue:

```python
# Note: Not closing figure here - it needs to remain open for later PNG conversion
# The figure will be garbage collected when no longer referenced
```

Matplotlib figures are heavyweight objects. In a long-running file monitor that
processes many beams, unclosed figures accumulate and consume memory. Python's garbage
collector may not reclaim them promptly because matplotlib maintains internal references.

**Recommendation:** Close figures explicitly after conversion in the Uploader:

```python
def _matplotlib_figure_to_png_bytes(self, fig) -> Optional[bytes]:
    try:
        img_bytes = io.BytesIO()
        fig.savefig(img_bytes, format='PNG', dpi=300, bbox_inches='tight')
        img_bytes.seek(0)
        return img_bytes.getvalue()
    finally:
        plt.close(fig)  # Always close after converting
```

### I-10. File readiness check is insufficient

**File:** `src/data_manipulation/file_monitoring/folder_monitor.py:118-146`
**Severity:** MEDIUM
**Category:** Reliability

The folder monitor uses a fixed 2-second sleep and then checks if `Results.csv` exists
and is non-empty. This is a race condition: on a slow network drive (which an iDrive
over USB often is), the file might exist and be non-empty but still be actively being
written to.

```python
time.sleep(2)  # Fixed delay - may not be enough for large transfers

if os.path.getsize(results_csv) == 0:
    return False
```

**Recommendation:** Check file stability (size hasn't changed over an interval):

```python
def _is_folder_ready(self, folder_path, stability_checks=3, check_interval=2):
    results_csv = os.path.join(folder_path, "Results.csv")
    if not os.path.exists(results_csv):
        return False

    prev_size = -1
    for _ in range(stability_checks):
        current_size = os.path.getsize(results_csv)
        if current_size == 0:
            return False
        if current_size == prev_size:
            return True  # Size stable
        prev_size = current_size
        time.sleep(check_interval)

    return False  # Size still changing
```

---

## 4. Minor Issues and Suggestions

### M-1. Python code uses Java-style getter/setter pattern instead of properties

**Files:** All model files in `src/data_manipulation/models/`
**Category:** Python Idioms

The Python models use `get_X()` / `set_X()` methods:

```python
def get_type(self):
    return self._type

def set_type(self, type_value):
    self._type = type_value
```

This is a Java/C# pattern. Pythonic style uses properties:

```python
@property
def type(self):
    return self._type

@type.setter
def type(self, value):
    self._type = value
```

This is a style preference, not a bug. No need to change it before the deadline, but
worth knowing for future Python projects. The getter/setter approach adds ~150 lines of
boilerplate across the model files.

### M-2. Duplicate getter/setter definitions in Geo6xfffModel

**File:** `src/data_manipulation/models/Geo6xfffModel.py:76-84 and 184-202`
**Category:** Code Duplication

The `Geo6xfffModel` defines `get_relative_output()`, `get_relative_uniformity()`, and
`get_center_shift()` twice: once at lines 77-84 (with `Decimal(str(value))` conversion)
and again at lines 185-202 (without conversion). The second set shadows the first,
meaning the Decimal conversion in the setter is bypassed:

```python
# Lines 78 (first definition - converts to Decimal)
def set_relative_output(self, value): self._relative_output = Decimal(str(value))

# Lines 199 (second definition - does NOT convert)
def set_relative_output(self, relative_output):
    self._relative_output = relative_output  # No Decimal conversion!
```

**Recommendation:** Remove the duplicate definitions at lines 184-202.

### M-3. Query parameter naming inconsistency between controllers

**Files:** `BeamsController.cs` vs `GeoCheckController.cs`

Beams uses camelCase query params: `?machineId=X`
GeoChecks uses kebab-case: `?machine-id=X`

```csharp
// BeamsController - camelCase
[FromQuery] string? machineId = null,

// GeoCheckController - kebab-case
[FromQuery(Name = "machine-id")] string? machineId = null,
```

**Recommendation:** Pick one convention. Since the JSON responses use camelCase
(configured in `Program.cs`), use camelCase for query parameters too.

### M-4. GetBeamTypes returns a hardcoded list

**File:** `src/api/Repositories/Supabase/SupabaseBeamRepository.cs:131-136`

```csharp
public async Task<IReadOnlyList<string>> GetBeamTypesAsync(...)
{
    var types = new[] { "6e", "9e", "12e", "16e", "10x", "15x", "6xff" };
    return await Task.FromResult(types.ToList().AsReadOnly());
}
```

This list doesn't match the types used in `DataProcessor.py` (`2.5x`, `6x`, `6xFFF`).
Consider querying `SELECT DISTINCT type FROM beams` instead.

### M-5. Error responses don't follow a consistent format

Controllers return different error shapes:

```csharp
return BadRequest("Month must be between 1 and 12.");           // plain string
return Conflict(exception.Message);                              // exception message
return StatusCode(500, "An error occurred...");                  // generic message
return NotFound($"Beam with id '{id}' was not found.");         // interpolated string
```

**Recommendation:** Use a consistent error response DTO:

```csharp
public record ApiError(string Message, string? Code = null);

return BadRequest(new ApiError("Month must be between 1 and 12.", "INVALID_MONTH"));
```

### M-6. The `test` methods in extractors and uploaders are production code

**Files:** `data_extractor.py`, `Uploader.py`, `DataProcessor.py`

Methods like `extractTest()`, `uploadTest()`, `RunTest()`, and `testeModelExtraction()`
are test helpers mixed into production classes. They call `logging.basicConfig()` (which
should only be called once at application startup) and add clutter.

**Recommendation:** Move test-specific code to the test directory, or use Python's
`unittest` / `pytest` framework. The `is_test` flag pattern creates two code paths
that can diverge over time.

### M-7. Approval errors are silently swallowed

**File:** `src/api/Controllers/BeamsController.cs:232-240`

```csharp
if (errors.Any())
{
    // If some failed, return 207 Multi-Status or just bad request with details?
    // For simplicity, we'll return Ok with the successful ones...
}
return Ok(results);  // Always returns 200 even if some approvals failed
```

The comment shows the team was aware of this. For a clinical system, partial failures
should be clearly communicated. Consider returning 207 Multi-Status with both successes
and failures.

### M-8. `logger.error` used for success messages

**File:** `src/data_manipulation/ETL/Uploader.py:754`

```python
logger.error(f"Uploaded {success_count}/{len(metrics)} baseline metric records")
```

This should be `logger.info`.

### M-9. README is essentially empty

**File:** `README.md`

```
Read me file
```

For an open-source clinical tool, the README should include: project description,
architecture diagram, setup instructions, required environment variables, how to run
tests, deployment notes, and a disclaimer about clinical use.

### M-10. No database migrations or schema management

There are no migration files or schema creation scripts. The database schema is only
documented in `GEOCHECK_API.md`. Anyone setting up the project has to manually create
tables.

**Recommendation:** Add a `schema.sql` file (or use a migration tool) so anyone can
reproduce the database schema.

### M-11. Missing `2.5x` beam type in C# API

The Python `DataProcessor.py` supports `2.5x` beams, but `GetBeamTypesAsync()` in the
C# API doesn't include `2.5x`, and neither does the beam_map include a corresponding
entry in the API's hardcoded type list. This means 2.5 MV beam data can be ingested
by Python but won't appear correctly in the frontend's beam type filter.

### M-12. Image DPI hardcoded

**File:** `src/data_manipulation/ETL/image_extractor.py:51`

```python
img = ArrayImage(normalized, dpi=280)
```

The DPI value of 280 is hardcoded. If the detector resolution changes (different machine
model, firmware update), this would need to change. Consider making it configurable or
extracting it from the XIM file metadata.

---

## 5. Clinical / Domain Correctness Assessment

### 5.1 Flatness and Symmetry Calculations: CORRECT (with caveats)

The team made the right decision to delegate flatness and symmetry calculations to
**pylinac** (v3.38.0) with `Protocol.VARIAN`:

```python
analysis = FieldAnalysis(img)
analysis.analyze(
    protocol=Protocol.VARIAN,
    in_field_ratio=0.8,
    edge_detection_method='FWHM',
)
```

- `Protocol.VARIAN` uses Varian's definition of flatness and symmetry, which is
  appropriate for a Varian TrueBeam linac.
- `in_field_ratio=0.8` (central 80%) is the standard measurement region.
- `FWHM` edge detection is a standard approach.
- The fallback to `in_field_ratio=0.5` when 0.8 fails is a reasonable defensive
  measure, but it **should be flagged in the stored data** so physicists know which
  analysis was used.

**Caveat:** The flatness/symmetry values are stored without indicating which protocol
or in-field ratio was used. If the fallback triggers, the physicist sees values computed
with different parameters than expected, with no indication.

**Recommendation:** Add a `analysis_notes` field to store which parameters were used.

### 5.2 Absolute Dose Conversion (DOC Factor): METHODOLOGY IS SOUND

The DOC factor approach (`DOC = MsdAbs / MpcRel`) is a standard method for converting
MPC relative output readings to absolute dose:

```
Absolute Output = MPC Relative Output * DOC Factor
```

The date-range management for DOC factors (auto-closing the previous factor's end date
when a new one is created) is well-implemented in `SupabaseDocFactorRepository.CreateAsync()`.

**Caveats:**
- Division by zero risk (addressed in C-3 above)
- No validation that MsdAbs comes from a calibrated measurement
- No audit trail for DOC factor changes
- The DOC factor is computed on the server but the actual absolute dose conversion
  (multiplying beam output by DOC factor) doesn't appear to be implemented anywhere
  in the current codebase — the factor is stored but never applied

### 5.3 MPC File Parsing: CORRECT

The CSV parsing in `data_extractor.py` correctly:
- Reads the standard MPC Results.csv format (`Name [Unit], Value, Threshold, ...`)
- Maps field names to model attributes using `'in' name` checks
- Handles all expected MPC check types (IsoCenterSize, BeamOutputChange, MLCLeaf, etc.)
- Extracts baseline status from `Check.xml` using proper XML namespace handling

**Caveat on leaf numbering:** The model stores 60 leaves (1-60) but the CSV parser
validates leaves 1-60. TrueBeam Millennium 120 MLCs have leaves 1-60 per bank, so this
is correct. However, for HD 120 MLCs the numbering might differ. Document the supported
MLC model.

### 5.4 Threshold Values: NEED DOCUMENTATION

The hardcoded thresholds in `ReportService.cs` should be traced to a clinical standard.
The values used (Relative Output ±2%, Relative Uniformity ±3%, Center Shift ≤2mm) are
reasonable for daily MPC checks based on typical institutional protocols, but they are
**not universal standards**. Different institutions may use different tolerances based on
their specific QA protocols.

**Recommendation:** Add a configuration file or database table documenting:
- What standard or protocol each threshold is based on
- Who approved the threshold values (the clinic's medical physicist)
- When they were last reviewed

---

## 6. Positive Callouts

### P-1. Excellent use of the Repository Pattern with Interface Abstractions

The `IBeamRepository` / `SupabaseBeamRepository` / `InMemoryBeamRepository` pattern is
textbook-clean separation of concerns. The in-memory implementations enable testing
without a database, and the `ServiceCollectionExtensions.cs` factory methods make
swapping implementations trivial. This is genuinely well-done architecture.

### P-2. Smart delegation to pylinac for physics calculations

Rather than implementing flatness and symmetry calculations from scratch (which would
be error-prone and hard to validate), the team correctly used pylinac — a well-tested,
peer-reviewed library specifically designed for medical physics QA. The choice of
`Protocol.VARIAN` and `in_field_ratio=0.8` shows they understood the clinical context.

### P-3. Image processing pipeline is well-structured

The dark field subtraction, flood field normalization pipeline in `image_extractor.py`
follows the correct physics:

```
corrected_clinical = clinical - dark
corrected_flood = flood - dark
normalized = corrected_clinical / corrected_flood
```

This is the standard approach for flat-field correction in detector-based beam
measurements. The zero-division protection with threshold clamping is appropriate.

### P-4. DOC factor date range management is thoughtful

The `SupabaseDocFactorRepository.CreateAsync()` method automatically adjusts the end
date of the previous DOC factor when a new one is created. This ensures there are no
gaps or overlaps in the validity periods, which is important for correct dose
calculations across time.

### P-5. Proper use of CancellationToken throughout the API

Every async method in the C# API accepts and respects `CancellationToken`. This is a
best practice that many professional projects skip. It means long-running requests can
be properly cancelled when clients disconnect.

### P-6. Comprehensive geometry check data model

The `Geo6xfffModel` and corresponding database schema capture the full breadth of MPC
geometry checks: isocenter, gantry, couch, collimation, MLC leaves, MLC backlash, jaws,
and jaw parallelism. This is thorough and matches the Varian MPC output format well.

### P-7. Good separation between data pipeline and API

The Python data pipeline (file monitoring, extraction, upload) and the C# API are
properly decoupled. They communicate only through the shared database, which means
either can be modified or replaced independently. This is a sound architectural decision
for a system that needs to run reliably in a clinical environment.

### P-8. Unit tests use proper patterns

The xUnit tests use:
- Arrange-Act-Assert pattern
- `[Theory]` with `[InlineData]` for parameterized tests
- Moq for dependency mocking
- FluentAssertions for readable assertions
- Proper boundary value testing (e.g., month 0, month 13, year 1899)

---

## Summary Prioritization Matrix

| Priority | Issue | Effort | Impact |
|----------|-------|--------|--------|
| **Week 1** | C-1: Authentication on approval endpoints | High | Critical for clinical use |
| **Week 1** | C-2: Implement DetermineCheckStatus | Medium | Critical for correctness |
| **Week 1** | C-3: DOC factor division-by-zero | Low | Critical for safety |
| **Week 1** | C-4: Fix -1 sentinel values in parser | Low | Critical for data integrity |
| **Week 2** | I-1: Fix folder_monitor.py syntax error | Low | High (app won't start) |
| **Week 2** | I-2: Fix dead code in geoModelUpload | Low | High (will crash) |
| **Week 2** | C-5: Reconcile threshold sources | Medium | Critical for consistency |
| **Week 2** | I-3: Single Supabase client | Medium | Performance |
| **Week 3** | I-4: Replace Console.WriteLine | Low | Observability |
| **Week 3** | I-7: Beam type detection ordering | Low | Correctness |
| **Week 3** | I-8: Configurable CORS | Low | Production readiness |
| **Week 4** | M-9: README documentation | Medium | Open-source readiness |
| **Week 4** | M-10: Schema migration file | Medium | Deployment |
| **Backlog** | I-5, I-6, I-9, I-10 | Various | Reliability improvements |
| **Backlog** | M-1 through M-8 | Various | Polish |

---

*This review is intended to be constructive. The team has built a functional, well-organized system in a
challenging medical physics domain. The critical issues identified are standard gaps in any student project
that hasn't yet gone through a security and clinical review — they are entirely fixable within the remaining
timeline. Focus on the safety-critical items first (authentication, correct status calculation, input validation)
and the system will be in strong shape for clinical evaluation.*
