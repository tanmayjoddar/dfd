# FRBAS Master Technical Defense & Architectural Manual (Big Brother Edition)

This manual provides a file-by-file descriptive breakdown of the Facial Recognition Based Attendance System (FRBAS). For every participating file in each operational segment, this guide explains:
1. **File Role**: What this file does in the architecture.
2. **Descriptive Summary**: What happens inside this file for this specific feature.
3. **Essential Working Code**: The exact code snippet to quote and defend.
4. **Connection to Next File**: How data or execution hands over to the next file in the pipeline.

---

## Segment 1: Gate In & Gate Out (Web Browser Kiosk)

```
[routes/web.php] 
       │ (GET /gate/in, GET /gate/out, POST /gate/capture)
       ▼
[GateTerminalController.php] ──► [Device.php] (Terminal metadata)
       │ (extracts hex from image)
       ▼
[BiometricApiService.php] ──► [NIC Cloud API /identify-1n] (Returns student_id)
       │
       ▼
[AttendanceStudent.php] (Checks is_identifiable_1n == true)
       │
       ▼
[AttendanceCalculationService.php] (Checks 60s debounce in attendance_events)
       │
       ├─► [AttendanceEvent.php] (Logs verified or rate_limited)
       ├─► [AttendanceRecord.php] (Consolidates daily punch)
       ▼
[terminal.blade.php] (Renders result, 2.5s rejection latch, 3s auto-scan poller)
```

---

### 1.1 routes/web.php
- **File Role**: System routing table for web requests.
- **Descriptive Summary**: Defines the public endpoints for the physical entrance and exit terminals, mapping them to `GateTerminalController`.
- **Essential Working Code**:
  ```php
  // Lines 242-244
  Route::get('/gate/in', [App\Http\Controllers\GateTerminalController::class, 'showIn'])->name('gate.in');
  Route::get('/gate/out', [App\Http\Controllers\GateTerminalController::class, 'showOut'])->name('gate.out');
  Route::post('/gate/capture', [App\Http\Controllers\GateTerminalController::class, 'capture'])->name('gate.capture');
  ```
- **Connection to Next File**: When a browser navigates to `/gate/in` or `/gate/out`, Laravel invokes `GateTerminalController::showIn()` or `GateTerminalController::showOut()`. When the camera triggers a scan, it sends an AJAX POST request to `/gate/capture`, invoking `GateTerminalController::capture()`.

---

### 1.2 app/Http/Controllers/GateTerminalController.php
- **File Role**: Primary kiosk controller. Enforces turnstile direction, orchestrates cloud identification, gatekeeps access, and suppresses duplicate swipes.
- **Descriptive Summary**:
  - `showIn()` and `showOut()` resolve the terminal from the `Device` model and inject directional themes (emerald green for In, crimson red for Out) into `terminal.blade.php`.
  - `capture()` extracts the face hex embedding, delegates biometric matching to `BiometricApiService`, queries `AttendanceStudent` by primary key, verifies `is_identifiable_1n`, checks 60-second debounce via `AttendanceCalculationService`, and logs to `AttendanceEvent`.
- **Essential Working Code**:
  ```php
  // Lines 124-142: Query Cloud and Lookup Student
  $result = $this->biometricService->identifyFace1N($faceHex);

  if (!$result['success']) {
      AttendanceEvent::create([
          'attendance_student_id' => null,
          'status'                => 'unidentified',
          'direction'             => $direction,
          'failure_reason'        => $result['message'],
          'captured_at'           => $now,
      ]);
      return response()->json(['success' => false, 'message' => $result['message']], 404);
  }

  // Look up student ONLY by primary key returned by cloud
  $student = AttendanceStudent::find($result['student_id']);

  // Lines 143-157: Gatekeeper Check
  if (!$student->is_identifiable_1n) {
      AttendanceEvent::create([
          'attendance_student_id' => $student->id,
          'device_id'             => $deviceId,
          'match_type'            => '1:N',
          'direction'             => $direction,
          'status'                => 'rejected_unregistered_1n',
          'failure_reason'        => 'Student is not registered for 1:N Gate Attendance.',
          'captured_at'           => $now,
      ]);
      return response()->json(['success' => false, 'message' => 'Student is not registered for 1:N Gate Attendance.'], 403);
  }

  // Lines 203-225: Debounce Suppression
  $remainingSeconds = $this->calcService->getDebounceRemainingSeconds($student->id, $direction, $deviceId);
  if ($remainingSeconds !== null && $remainingSeconds > 0) {
      AttendanceEvent::create([
          'attendance_student_id' => $student->id,
          'device_id'             => $deviceId,
          'match_type'            => '1:N',
          'direction'             => $direction,
          'status'                => 'rate_limited',
          'failure_reason'        => "Rapid-fire duplicate punch suppressed (cooldown active: {$remainingSeconds}s remaining).",
          'captured_at'           => $now,
      ]);
      return response()->json([
          'success'            => true,
          'status'             => 'rate_limited',
          'cooldown_remaining' => $remainingSeconds,
          'message'            => "Punch already logged recently for {$student->name} (Debounced - {$remainingSeconds}s cooldown active).",
      ], 200);
  }
  ```
- **Connection to Next File**: Passes the raw face hex to `BiometricApiService::identifyFace1N()`. After receiving the student ID, queries `AttendanceStudent`, calls `AttendanceCalculationService`, writes to `AttendanceEvent` and `AttendanceRecord`, and returns JSON to `terminal.blade.php`.

---

### 1.3 app/Services/BiometricApiService.php
- **File Role**: Gateway to external NIC Cloud Biometric Microservices.
- **Descriptive Summary**: Serializes the 512-dimensional face embedding and dispatches an HTTP POST request to the cloud endpoint `/identify-1n`. Evaluates candidate similarity scores against threshold `0.65` and applies ambiguity filtering (< 0.08 margin).
- **Essential Working Code**:
  ```php
  // Lines 180-215
  public function identifyFace1N(string $faceHex): array
  {
      $response = Http::timeout(10)->post($this->cloudBaseUrl . '/identify-1n', [
          'hex'       => $faceHex,
          'threshold' => 0.65,
          'top_k'     => 2,
      ]);

      $data = $response->json();
      $candidates = $data['matches'] ?? [];

      if (empty($candidates)) {
          return ['success' => false, 'message' => 'Face not matching. Please try again or register your face.'];
      }

      if (count($candidates) >= 2) {
          $margin = $candidates[0]['similarity'] - $candidates[1]['similarity'];
          if ($margin < 0.08) {
              return ['success' => false, 'status' => 'ambiguous', 'message' => 'Ambiguous face detected.'];
          }
      }

      return [
          'success'    => true,
          'status'     => 'verified',
          'student_id' => (int) $candidates[0]['student_id'],
          'similarity' => (float) $candidates[0]['similarity'],
          'candidates' => $candidates,
      ];
  }
  ```
- **Connection to Next File**: Returns an associative array containing `student_id` back to `GateTerminalController.php`.

---

### 1.4 app/Models/AttendanceStudent.php
- **File Role**: Eloquent model representing the intern master record.
- **Descriptive Summary**: Contains student metadata, the 6000+ hex character embedding column, and the boolean access control flag `is_identifiable_1n`.
- **Essential Working Code**:
  ```php
  // Lines 16-24
  protected $fillable = [
      'name', 'roll_id', 'department', 'image_path', 'embedding', 'is_identifiable_1n'
  ];

  protected $casts = [
      'is_identifiable_1n' => 'boolean',
  ];
  ```
- **Connection to Next File**: `GateTerminalController.php` checks `is_identifiable_1n`. If true, it calls `AttendanceCalculationService` to verify debounce.

---

### 1.5 app/Services/AttendanceCalculationService.php
- **File Role**: Real-time business logic engine for time debouncing and shift interval consolidation.
- **Descriptive Summary**: Queries `attendance_events` for recent verified punches to calculate cooldown time, and pairs IN and OUT events in Asia/Kolkata timezone to calculate total daily work hours.
- **Essential Working Code**:
  ```php
  // Lines 27-50
  public function getDebounceRemainingSeconds(int $studentId, string $direction, ?int $deviceId = null): ?int
  {
      $cutoff = Carbon::now('Asia/Kolkata')->subSeconds($this->debounceSeconds); // 60s

      $query = AttendanceEvent::where('attendance_student_id', $studentId)
          ->where('direction', $direction)
          ->where('status', 'verified')
          ->where('captured_at', '>=', $cutoff)
          ->orderBy('captured_at', 'desc');

      if ($deviceId !== null) {
          $query->where('device_id', $deviceId);
      }

      $lastEvent = $query->first();
      if (!$lastEvent) {
          return null;
      }

      $eventTime = Carbon::parse($lastEvent->captured_at, 'Asia/Kolkata');
      $elapsed = Carbon::now('Asia/Kolkata')->diffInSeconds($eventTime);
      $remaining = $this->debounceSeconds - $elapsed;

      return $remaining > 0 ? (int) $remaining : null;
  }
  ```
- **Connection to Next File**: Supplies remaining cooldown seconds or daily duration summary back to `GateTerminalController.php`, which writes events to `AttendanceEvent` and `AttendanceRecord`.

---

### 1.6 app/Models/AttendanceEvent.php and app/Models/AttendanceRecord.php
- **File Role**: Database models for audit logs and consolidated daily timesheets.
- **Descriptive Summary**:
  - `AttendanceEvent`: High-volume audit table. Every camera capture creates a row here (`verified`, `rate_limited`, `rejected_unregistered_1n`, `unidentified`).
  - `AttendanceRecord`: Consolidated daily attendance row. One row per student per day storing `check_in_at` and `check_out_at`. Rate-limited duplicate swipes never touch this table.
- **Essential Working Code**:
  ```php
  // AttendanceEvent creation
  AttendanceEvent::create([
      'attendance_student_id' => $student->id,
      'device_id'             => $deviceId,
      'match_type'            => '1:N',
      'direction'             => $direction,
      'status'                => 'verified', // or 'rate_limited'
      'captured_at'           => $now,
  ]);

  // AttendanceRecord update
  $record->update(['check_out_at' => $now]);
  ```
- **Connection to Next File**: Database state is committed, and JSON response is sent to `terminal.blade.php`.

---

### 1.7 resources/views/gate/terminal.blade.php
- **File Role**: Client-side kiosk user interface and asynchronous state machine.
- **Descriptive Summary**:
  - Displays live video stream.
  - Automatically or manually triggers face capture via `triggerScan()`.
  - Implements **2.5s rejection latch** (`showError`) so error messages remain readable.
  - Implements **3s poller** (`toggleAutoScan`) that pauses whenever a card is active.
  - Implements **4s debounce latch** (`showSuccess`) showing amber `COOLDOWN ACTIVE` banner.
- **Essential Working Code**:
  ```javascript
  // Lines 522-566
  function showError(msg) {
      document.getElementById('idleBox').style.display = 'none';
      const eBox = document.getElementById('errorBox');
      eBox.style.display = 'block';
      document.getElementById('errMsg').textContent = msg;

      setTimeout(() => {
          eBox.style.display = 'none';
          document.getElementById('idleBox').style.display = 'block';
          document.getElementById('gateStatusMsg').textContent = 'Ready for next scan.';
      }, 2500); // 2.5 second hold
  }

  function toggleAutoScan(el) {
      if (el.checked) {
          autoScanInterval = setInterval(() => {
              const verifiedVisible = document.getElementById('verifiedBox').style.display !== 'none';
              const errorVisible = document.getElementById('errorBox').style.display !== 'none';
              if (!isProcessing && !verifiedVisible && !errorVisible) {
                  triggerScan();
              }
          }, 3000); // 3 second poll
      } else {
          clearInterval(autoScanInterval);
      }
  }
  ```

---

## Segment 2: Mounted Hardware Turnstiles (DeviceCaptureController)

```
[Mounted Physical Camera] (Hikvision / Dahua / Jetson)
       │ (HTTP POST /api/attendance/capture)
       │ (Headers: X-Device-Code, X-Device-Token)
       ▼
[routes/api.php]
       │
       ▼
[DeviceCaptureController.php] ──► [Device.php] (Validates API key & direction)
       │
       ▼
[BiometricApiService.php] (Calls Cloud /identify-1n)
       │
       ▼
[AttendanceStudent.php] & [FrbasRegistration.php] (Student-only check)
       │
       ▼
[AttendanceCalculationService.php] (Debounce check for device_id)
       │
       ├─► [AttendanceEvent.php] (Logs punch event)
       └─► [AttendanceRecord.php] (Consolidates punch)
       ▼
[Hardware Turnstile Relay] (Receives HTTP 200, fires dry contact relay)
```

---

### 2.1 routes/api.php
- **File Role**: API routing entry point for external IoT devices.
- **Descriptive Summary**: Exposes stateless endpoint `/api/attendance/capture` exempt from web CSRF tokens.
- **Essential Working Code**:
  ```php
  // Line 16
  Route::post('/attendance/capture', [DeviceCaptureController::class, 'capture']);
  ```
- **Connection to Next File**: Hardware requests hit this route and are dispatched to `DeviceCaptureController::capture()`.

---

### 2.2 app/Models/Device.php
- **File Role**: Model representing physical gate turnstile terminals.
- **Descriptive Summary**: Stores terminal configuration: unique `device_code`, `api_key` bearer token, enforced `direction`, and boolean `is_active`.
- **Essential Working Code**:
  ```php
  protected $fillable = [
      'device_code', 'name', 'direction', 'location', 'ip_address', 'api_key', 'is_active'
  ];
  ```
- **Connection to Next File**: `DeviceCaptureController.php` queries this model using headers `X-Device-Code` and `X-Device-Token`.

---

### 2.3 app/Http/Controllers/DeviceCaptureController.php
- **File Role**: Hardware API controller. Handles machine-to-machine authentication, direction enforcement, biometric identification, staff restriction, debounce, and relay triggers.
- **Descriptive Summary**:
  - Authenticates device credentials from HTTP headers against `Device` model.
  - Pulls pre-configured direction from `$device->direction`.
  - Calls `BiometricApiService::identifyFace1N()`.
  - Checks `is_identifiable_1n` and student-only policy (`rejected_staff`).
  - Calls `AttendanceCalculationService` for debounce.
  - Returns HTTP 200 `{ "success": true, "status": "verified" }` so the hardware relay trips.
- **Essential Working Code**:
  ```php
  // Lines 35-52: Device Authentication
  $deviceCode = $request->header('X-Device-Code', $request->input('device_code'));
  $token = $request->header('X-Device-Token', $request->input('api_key'));

  $device = Device::where('device_code', $deviceCode)->first();
  if (!$device || !$device->is_active) {
      return response()->json(['success' => false, 'message' => 'Unauthorized device.'], 401);
  }

  $direction = $device->direction ?? $request->input('direction', 'in');

  // Lines 128-158: Student-Only Policy Enforcement
  if ($student->frbasRegistration && $student->frbasRegistration->type !== 'intern') {
      AttendanceEvent::create([
          'attendance_student_id' => $student->id,
          'device_id'             => $device->id,
          'match_type'            => '1:N',
          'direction'             => $direction,
          'status'                => 'rejected_staff',
          'failure_reason'        => 'Gate terminal attendance is reserved for Students only.',
          'captured_at'           => $capturedAt,
      ]);
      return response()->json(['success' => false, 'message' => 'Staff attendance restricted.'], 403);
  }

  // Lines 188-215: Verified Punch Confirmation
  AttendanceEvent::create([
      'attendance_student_id' => $student->id,
      'device_id'             => $device->id,
      'match_type'            => '1:N',
      'direction'             => $direction,
      'status'                => 'verified',
      'similarity_score'      => $result['similarity'],
      'captured_at'           => $capturedAt,
  ]);

  return response()->json([
      'success' => true,
      'status'  => 'verified',
      'student' => ['id' => $student->id, 'name' => $student->name],
  ], 200);
  ```
- **Connection to Next File**: Returns JSON to the physical turnstile controller. The device firmware detects `success: true` and activates its GPIO relay to open the turnstile.

---

## Segment 3: Complete 1:N Biometric Flow & Dual Storage

```
[User Form Save Changes]
       │
       ▼
[AttendanceController.php] (Profile update)
       │
       ├─► [attendance_students.embedding] (Stored locally: 6000+ hex chars)
       └─► [BiometricApiService.php] (Calls Cloud /register-1n)
                 │
                 ▼
          [attendance_students.is_identifiable_1n = true] (Auto-synced)

─────────────────────────────────────────────────────────────────

[1:N Public Kiosk / Console] (resources/views/frbas/public-identify-1n.blade.php)
       │ (POST /attendance/1n/identify)
       ▼
[Attendance1NController.php]
       │ (Calls BiometricApiService::identifyFace1N)
       ▼
[AttendanceCalculationService.php] (Calculates duration & debounce)
       │
       ▼
[AttendanceEvent.php] & [AttendanceRecord.php] (Persists punch records)
```

---

### 3.1 app/Http/Controllers/AttendanceController.php
- **File Role**: Student profile management and biometric enrollment synchronization.
- **Descriptive Summary**: When an intern profile is saved, it writes the vector embedding locally to `attendance_students.embedding` (dual storage) and dispatches it to the NIC Cloud gallery index via `BiometricApiService::registerFace1N()`. Once the cloud confirms, it flips `is_identifiable_1n = true`.
- **Essential Working Code**:
  ```php
  // Lines 318-324
  try {
      $biometricService = app(\App\Services\BiometricApiService::class);
      $regRes = $biometricService->registerFace1N((string) $studentToSync->id, (string) $studentToSync->embedding);
      
      // Auto-sync 1:N access in database
      $studentToSync->update(['is_identifiable_1n' => true]);
      Log::info('1:N external face registration synced on profile save', ['student_id' => $studentToSync->id]);
  } catch (\Exception $e) {
      Log::warning('1:N sync deferred: ' . $e->getMessage());
  }
  ```
- **Connection to Next File**: Enables `is_identifiable_1n` in the database, allowing `Attendance1NController.php` and `GateTerminalController.php` to authorize this student.

---

### 3.2 app/Http/Controllers/Attendance1NController.php
- **File Role**: Controller for the dedicated 1:N identification portal and public kiosks.
- **Descriptive Summary**: Handles `/attendance/1n/identify`. Auto-detects swipe direction (first swipe today = `in`, subsequent swipe = `out`), checks debounce, records verified punch in `AttendanceEvent`, consolidates with `AttendanceRecord`, and computes daily work duration.
- **Essential Working Code**:
  ```php
  // Lines 176-188, 230-255
  $existing = AttendanceRecord::where('attendance_student_id', $student->id)
      ->whereDate('created_at', $today)
      ->first();

  $direction = ($existing && $existing->check_in_at) ? 'out' : 'in';

  // Record verified event
  AttendanceEvent::create([
      'attendance_student_id' => $student->id,
      'match_type'            => '1:N',
      'direction'             => $direction,
      'status'                => 'verified',
      'similarity_score'      => $result['similarity'],
      'captured_at'           => $now,
  ]);

  // Update consolidated daily record
  if (!$existing) {
      $existing = AttendanceRecord::create([
          'attendance_student_id' => $student->id,
          'session_name'          => '1:N Terminal',
          'similarity'            => $result['similarity'],
          'check_in_at'           => $now,
      ]);
  } else if ($direction === 'out') {
      $existing->update([
          'check_out_at' => $now,
          'similarity'   => $result['similarity'],
      ]);
  }

  $dailySummary = $this->calcService->calculateDailySummary($student->id, $today);
  ```
- **Connection to Next File**: Returns JSON with plain Date and Time (`captured_date`, `captured_at`) to `resources/views/frbas/public-identify-1n.blade.php`.

---

## Segment 4: Work Domain & Qualification Domain (Soft Deletes)

```
[routes/web.php] (admin.domains.*)
       │
       ▼
[AdminDomainController.php]
       │
       ├─► [destroyWorkDomain($id)] ──► [WorkDomain.php] (Writes deleted_at timestamp)
       ├─► [restoreWorkDomain($id)] ──► [WorkDomain.php] (Resets deleted_at = null)
       │
       ▼
[FrbasRegistration.php] (Declares withTrashed() on relationships)
       │
       ▼
[admin/domains/index.blade.php] (Renders active and archived domain tables)
```

---

### 4.1 app/Http/Controllers/Admin/AdminDomainController.php
- **File Role**: Administrative domain lifecycle manager.
- **Descriptive Summary**: Provides full CRUD and restoration actions for both educational qualifications and project work domains. Soft-deletes records via `$domain->delete()` and restores archived records via `onlyTrashed()->findOrFail()->restore()`.
- **Essential Working Code**:
  ```php
  // Lines 137-154
  public function destroyWorkDomain($id)
  {
      $workDomain = WorkDomain::findOrFail($id);
      $name = $workDomain->name;
      $workDomain->delete(); // Sets deleted_at timestamp

      return redirect()->route('admin.domains.index')
          ->with('success', "Work Domain '{$name}' deleted from registration options.");
  }

  public function restoreWorkDomain($id)
  {
      $workDomain = WorkDomain::onlyTrashed()->findOrFail($id);
      $workDomain->restore(); // Resets deleted_at = null

      return redirect()->route('admin.domains.index')
          ->with('success', "Work Domain '{$workDomain->name}' restored to active list successfully.");
  }
  ```
- **Connection to Next File**: Updates database column `deleted_at` in table `work_domains`. `FrbasRegistration.php` uses `withTrashed()` to maintain relational integrity.

---

### 4.2 app/Models/FrbasRegistration.php
- **File Role**: Intern/staff registration profile model.
- **Descriptive Summary**: Configures relationships to `WorkDomain` and `QualificationDomain` with `withTrashed()`.
- **Essential Working Code**:
  ```php
  // Lines 35-43
  public function workDomain()
  {
      return $this->belongsTo(WorkDomain::class, 'work_domain_id')->withTrashed();
  }

  public function qualificationDomain()
  {
      return $this->belongsTo(QualificationDomain::class, 'qualification_domain_id')->withTrashed();
  }
  ```
- **Connection to Next File**: Ensures that admin reports, profile edit forms, and intern cards never fail with null errors when a domain is archived.

---

## Segment 5: FRBAS Access Control by Super Admin

```
[Incoming Web Request] (/frbas/* or /attendance/*)
       │
       ▼
[HasFrbasAccess.php] (Middleware)
       │
       ├─► $user->isAdmin() == true? ──► [Pass to Controller]
       │
       └─► $user->frbas_access == true?
                 ├─► Yes ──► [Pass to Controller]
                 └─► No  ──► [Redirect with error: "No access to FRBAS"]

─────────────────────────────────────────────────────────────────

[Admin Panel Toggle Button] ──► [AdminUserController::assignFrbas($id)]
                                      │
                                      ▼
                      [Update users.frbas_access boolean]
```

---

### 5.1 app/Http/Middleware/HasFrbasAccess.php
- **File Role**: Security gatekeeper middleware.
- **Descriptive Summary**: Intercepts requests targeting `/frbas/*` and `/attendance/*`. Super Admins bypass automatically. Regular users must have `users.frbas_access == true`. Unauthorized users are rejected and redirected.
- **Essential Working Code**:
  ```php
  // Lines 20-38
  public function handle(Request $request, Closure $next): Response
  {
      $user = auth()->user();

      // Admin always has full access to FRBAS
      if ($user->isAdmin()) {
          return $next($request);
      }

      // Regular users need the frbas_access flag set by admin
      if ($user->frbas_access) {
          if ($request->session()->get('auth_area') !== 'user') {
              return redirect()->route('login.show')->withErrors(['error' => 'Please login again.']);
          }
          return $next($request);
      }

      return redirect()->route('user.home')
          ->withErrors(['error' => 'You do not have access to FRBAS. Please contact your administrator.']);
  }
  ```
- **Connection to Next File**: If authorization passes, execution continues to `FrbasController` or `AttendanceController`.

---

### 5.2 app/Http/Controllers/Admin/AdminUserController.php
- **File Role**: Super Admin user administration controller.
- **Descriptive Summary**: Exposes `assignFrbas($id)` to toggle the `frbas_access` flag on any regular user account.
- **Essential Working Code**:
  ```php
  // Lines 411-422
  public function assignFrbas($id)
  {
      $user = User::where('role', 'user')->findOrFail($id);
      $user->update(['frbas_access' => !$user->frbas_access]);
      $action = $user->frbas_access ? 'granted' : 'revoked';

      return redirect()->back()->with('success', "FRBAS access {$action} for {$user->name}.");
  }
  ```
- **Connection to Next File**: Updates database column `users.frbas_access`, immediately altering access evaluation in `HasFrbasAccess.php`.

---

## Segment 6: Self-Registration vs. Admin Registration & Form Auto-Loading

```
[User loads /frbas/registration]
       │
       ▼
[FrbasController::registration] (Evaluates role & preloads user collections)
       │
       ▼
[resources/views/frbas/registration.blade.php]
       │
       ├─► If Developer/Officer: Executes loadMyProfileData() (Auto-fills self)
       └─► If Super Admin: Populates searchable dropdowns (Select user to edit)
       │
       ▼
[User Submits Form] ──► [AttendanceController::register]
                              │
                              ▼
           [Enforces Role Rules: Officers/Devs can only edit self]
```

---

### 6.1 app/Http/Controllers/FrbasController.php
- **File Role**: View controller for the FRBAS registration hub.
- **Descriptive Summary**: Evaluates the role permissions matrix (`canRegisterOfficer`, `canSelfRegisterDeveloper`, etc.). Pre-loads directory collections (`$allOfficers`, `$allDevelopers`, `$allInterns`) for Super Admin, or resolves supervising officer for Developers.
- **Essential Working Code**:
  ```php
  // Lines 52-94
  $canRegisterOfficer       = $user->isAdmin();
  $canSelfRegisterOfficer   = $isOfficer;
  $canRegisterDeveloper     = $user->isAdmin() || $isOfficer;
  $canRegisterIntern        = $user->isAdmin() || $isOfficer;
  $canSelfRegisterDeveloper = $isDeveloper;

  if ($user->isAdmin()) {
      $allOfficers = User::whereHas('stakeLevel', fn($q) => $q->where('name', 'NIC Officer'))->get();
      $allDevelopers = User::whereHas('stakeLevel', fn($q) => $q->where('name', 'NIC Developer'))->get();
      $allInterns = FrbasRegistration::where('type', 'intern')->with(['attendanceStudent'])->get();
  }
  ```
- **Connection to Next File**: Compacts permissions and collections into `resources/views/frbas/registration.blade.php`.

---

### 6.2 resources/views/frbas/registration.blade.php
- **File Role**: Interactive registration and profile editing blade form.
- **Descriptive Summary**:
  - For Developers/Officers: Displays "Load My Details" and triggers `loadMyProfileData()`, automatically injecting their name, email, and mobile into the form.
  - For Super Admin: Renders searchable dropdowns (`selectOfficer`, `selectDeveloper`, `selectIntern`). Selecting an individual triggers JavaScript hydration that fills the form, injects `editRecordId` and `editUserId`, loads their existing photo preview, and sets the submit button to "Save Changes & Update Face".
- **Essential Working Code**:
  ```javascript
  // Lines 1409-1430
  if (isSelfOfficer || isSelfDeveloper) {
      loadMyProfileData(); // Auto-fill personal details
  } else if (isIntern) {
      document.getElementById('regName').value = '';
      document.getElementById('regPhone').value = '';
      document.getElementById('regEmail').value = '';
  }
  ```
- **Connection to Next File**: Posts form data to `/attendance/register` handled by `AttendanceController::register()`.

---

### 6.3 app/Http/Controllers/AttendanceController.php
- **File Role**: Registration processor and role policy enforcer.
- **Descriptive Summary**: Validates input data, normalizes 10-digit Indian phone numbers, checks for duplicates, and strictly enforces role restrictions (e.g. Developers can only edit their own profile).
- **Essential Working Code**:
  ```php
  // Lines 116-141
  if ($type === 'nic_officer') {
      if (!$user->isAdmin() && !$this->isOfficer($user)) {
          abort(403, 'You cannot register or edit officers.');
      }
      if ($this->isOfficer($user) && !$user->isAdmin()) {
          if ($normalizedPhone !== $user->phone) {
              abort(403, 'NIC Officers can only edit their own profile.');
          }
      }
  }

  if ($type === 'nic_developer') {
      if (!$user->isAdmin() && !$this->isOfficer($user) && !$this->isDeveloper($user)) {
          abort(403, 'You cannot register or edit developers.');
      }
      if ($this->isDeveloper($user) && !$user->isAdmin() && !$this->isOfficer($user)) {
          if ($normalizedPhone !== $user->phone) {
              abort(403, 'Developers can only edit their own profile.');
          }
      }
  }

  if ($type === 'intern' && !$user->isAdmin() && !$this->isOfficer($user)) {
      abort(403, 'You cannot register or edit interns.');
  }
  ```
- **Connection to Next File**: If validation and authorization pass, it creates or updates `FrbasRegistration` and writes forensic logs to `FrbasAuditLog`.

---

## Segment 7: FRBAS Edit Section & Forensic Audit Logging

```
[AttendanceController::register]
       │
       ▼
Detects Edit: ($isEdit = $existingFrbas !== null)
       │
       ├─► Captures $oldValues (Current database state)
       ├─► Executes update on FrbasRegistration & AttendanceStudent
       ├─► Captures $newValues (Updated database state)
       │
       ▼
[FrbasAuditLog::create] (Writes immutable record to frbas_audit_logs)
       │
       ▼
[FrbasController::getAuditLogs] ──► [frbas/audit-logs.blade.php] (Forensic UI)
```

---

### 7.1 app/Http/Controllers/AttendanceController.php (Audit Trail Pipeline)
- **File Role**: Captures forensic snapshots during profile modifications.
- **Descriptive Summary**:
  - Detects edit mode if `edit_record_id` is supplied or if unique phone matches existing record.
  - Builds associative array `$oldValues` before mutation.
  - Applies updates to `FrbasRegistration` and `AttendanceStudent`.
  - Builds associative array `$newValues`.
  - Dispatches `FrbasAuditLog::create()` containing editor identity, IP address, user agent, before/after JSON diffs, and `photo_updated` boolean flag.
- **Essential Working Code**:
  ```php
  // Lines 340-371
  $oldValues = [
      'name'                    => $existingFrbas->name,
      'email'                   => $existingFrbas->email,
      'phone'                   => $existingFrbas->phone,
      'designation'             => $existingFrbas->designation,
      'gender'                  => $existingFrbas->gender,
      'qualification_domain_id' => $existingFrbas->qualification_domain_id,
      'work_domain_id'          => $existingFrbas->work_domain_id,
      'under_officer_id'        => $existingFrbas->under_officer_id,
  ];

  // Write immutable audit log
  FrbasAuditLog::create([
      'frbas_registration_id' => $existingFrbas->id,
      'user_id'               => $linkedUser?->id,
      'target_name'           => $existingFrbas->name,
      'target_type'           => $existingFrbas->type,
      'target_phone'          => $existingFrbas->phone,
      'target_email'          => $existingFrbas->email,
      'action'                => $photoUpdated ? 'updated_record_and_photo' : 'updated_details',
      'edited_by_user_id'     => $user->id,
      'editor_name'           => $user->name,
      'editor_email'          => $user->email,
      'editor_role'           => $user->isAdmin() ? 'Super Admin' : (method_exists($user, 'isOfficer') && $user->isOfficer() ? 'NIC Officer' : 'User'),
      'ip_address'            => $request->ip(),
      'user_agent'            => $request->userAgent(),
      'old_values'            => $oldValues,
      'new_values'            => $newValues,
      'photo_updated'         => $photoUpdated,
  ]);
  ```
- **Connection to Next File**: Persists row to `frbas_audit_logs` table, accessible via `FrbasAuditLog` model.

---

### 7.2 app/Models/FrbasAuditLog.php
- **File Role**: Eloquent model for the `frbas_audit_logs` table.
- **Descriptive Summary**: Defines fillable forensic fields and automatically casts `old_values` and `new_values` to JSON arrays.
- **Essential Working Code**:
  ```php
  // Lines 11-34
  protected $fillable = [
      'frbas_registration_id', 'user_id', 'target_name', 'target_type',
      'target_phone', 'target_email', 'action', 'edited_by_user_id',
      'editor_name', 'editor_email', 'editor_role', 'ip_address',
      'user_agent', 'old_values', 'new_values', 'photo_updated',
  ];

  protected $casts = [
      'old_values'    => 'array',
      'new_values'    => 'array',
      'photo_updated' => 'boolean',
  ];
  ```
- **Connection to Next File**: Queried by `FrbasController::getAuditLogs()` and displayed in `resources/views/frbas/audit-logs.blade.php`.

---

### 7.3 resources/views/frbas/audit-logs.blade.php
- **File Role**: Audit trail administrative viewer.
- **Descriptive Summary**: Renders cards detailing who modified which profile, the editor's IP address, timestamp, and visual comparison tables showing old values vs new values.
- **Connection**: Consumes paginated `FrbasAuditLog` collection provided by `FrbasController`.
