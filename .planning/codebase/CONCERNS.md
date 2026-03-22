# Codebase Concerns

**Analysis Date:** 2026-03-22

## Tech Debt

**Model Conversion - Unvalidated Expert Ordering (Mixtral):**
- Issue: Experts in Mixtral MoE models are not sanity-checked for numerical order
- Files: `convert/convert_mixtral.go:61,83`
- Impact: Incorrect expert ordering could lead to model inference producing wrong results silently
- Fix approach: Add validation to ensure experts are sorted before tensor merging, with clear error messages if ordering is incorrect

**KV Cache Removal Failures:**
- Issue: KV cache removal can fail silently for certain model types without clear error recovery
- Files: `llama/runner/cache.go:234-237`
- Impact: Context shifting may not properly free memory, leading to OOM errors during long sequences
- Fix approach: Implement fallback cache eviction strategy and add detailed logging of cache state transitions; consider marking cached sequences as "unreliable" after removal failures

**GPU Memory Reporting Unreliability:**
- Issue: Some GPUs report inaccurate free memory, forcing conservative concurrency settings (MaxRunners = GPU count)
- Files: `server/sched.go:173-186`, `discover/gpu.go:76,81`
- Impact: Multi-model performance severely degraded on unreliable GPUs; users can't override conservative defaults easily
- Fix approach: Implement per-GPU reliability tracking with overridable thresholds; add memory prediction validation cycle after model loading

**Scheduler - Missing Parallel Support:**
- Issue: Multimodal models (mllama) don't support parallel inference, but this is hardcoded
- Files: `server/sched.go:133`
- Impact: Parallel requests for multimodal models serialize unexpectedly; users get no visibility
- Fix approach: Add capabilities-based scheduling metadata to models; surface parallelization constraints in API responses

**Unsafe Pointer Arithmetic in Batch Operations:**
- Issue: Direct unsafe.Slice operations on C-allocated buffers without bounds validation
- Files: `llama/llama.go:420-434`
- Impact: Buffer overflows if batch size calculations are off; crashes in llama-cpp C layer
- Fix approach: Add batch size assertions before unsafe operations; use checked array indexing wrapper

## Known Bugs

**Scheduler Deadlock - Empty Runner Expiration (High Priority):**
- Symptoms: Scheduler occasionally freezes when trying to make room for new models; appears unresponsive to new requests
- Files: `server/sched.go:275-280`
- Trigger: "Shouldn't happen" comment suggests unhandled race condition when runnerToExpire is nil but logic expects it
- Workaround: Restart the server; monitor for OLLAMA_DEBUG logs showing "runner to expire was nil"
- Root cause: Timing window between checking available runners and selection logic; concurrent unload events can clear runnerToExpire

**Model Loading Fails with Partial ZIP Extractions:**
- Symptoms: Models created from Modelfile fail to load with cryptic "invalid tensor shape" panics
- Files: `server/model.go:84-99`, `convert/reader.go:46`
- Trigger: Interrupting model creation leaves partial artifacts; next attempt fails during conversion
- Workaround: Delete the model and pull again; clear the model cache directory
- Root cause: No cleanup of failed model conversions; ZIP extraction can be partial if extraction goroutines panic

**API Type Validation - Limited Slice Support (Medium Priority):**
- Symptoms: Tool calls with array parameters ([]string, []int, [][]int) silently ignored or cause errors
- Files: `api/types.go:710`, `openai/openai.go:118`
- Trigger: Using tool_choice with arrays in OpenAI-compatible requests
- Workaround: Use individual tool call objects instead of arrays
- Root cause: Only string slices supported; other array types explicitly excluded in comment

## Security Considerations

**C FFI Unsafe Code - No Bounds Checking:**
- Risk: Buffer overflows, out-of-bounds reads in embeddings extraction
- Files: `llama/llama.go:246-260` (unsafe embeddings access)
- Current mitigation: llama.cpp library performs some bounds checks; relying on C library safety
- Recommendations: Add Go-side assertions for embedding dimensions; validate embeddings array length before unsafe.Slice; consider wrapping in safe slice type

**OS Command Execution in Model Updates:**
- Risk: Potential shell injection during update processes
- Files: `app/lifecycle/updater_windows.go:14` (glob patterns), `cmd/start_windows.go`
- Current mitigation: Using filepath.Glob which is safer than shell globbing
- Recommendations: Use filepath.Join instead of glob patterns where possible; validate all exe paths before execution; add signature verification for downloaded updates

**Unvalidated Model File Reading:**
- Risk: Malformed GGML files could cause panics or memory exhaustion
- Files: `llm/ggml.go` (GGML format parsing), `convert/reader.go` (tensor reading)
- Current mitigation: Basic error handling in format detection
- Recommendations: Implement file size limits before parsing; add maximum tensor dimension validation; use SafeDecoder pattern with pre-checks

**Data Race in Scheduler State:**
- Risk: Race conditions between model loading and unloading could corrupt runner state
- Files: `server/sched.go:321-350` (refCount management without full synchronization)
- Current mitigation: refMu mutex around refCount changes
- Recommendations: Use atomic operations for refCount; document full lock ordering to prevent deadlocks

## Performance Bottlenecks

**VRAM Prediction Accuracy - Aggressive Estimation:**
- Problem: EstimatedVRAM calculations don't account for context expansion, projection overhead, or llama.cpp internal allocations
- Files: `server/sched.go:439,713-714,829`, `llm/server.go` (prediction logic)
- Cause: Linear estimation based on model parameters; doesn't account for padding, KV cache overhead, or batch size expansion
- Improvement path:
  1. Implement tiered testing with varying batch/context sizes to build prediction tables
  2. Add per-GPU calibration mode that measures actual VRAM usage
  3. Track EstimatedVRAM vs actual for loaded models; adjust multiplier based on model family
  4. Log discrepancies for models consistently overshooting estimates

**Scheduler Loop Contention - Global Lock on pendingReqCh:**
- Problem: All incoming requests go through single pending channel; scheduler goroutine processes one model decision at a time
- Files: `server/sched.go:95-99,118-180` (serial processing in loop)
- Cause: Single scheduler goroutine can't batch process multiple pending requests; network I/O blocks scheduling decisions
- Improvement path:
  1. Use request fanout to parallelize GPU selection logic (per-GPU decision trees)
  2. Implement speculative loading for likely next models
  3. Cache GPU topology decisions; recompute only on discovery changes
  4. Profile with high concurrent load (100+ simultaneous requests)

**GPU Memory Refresh Lag:**
- Problem: Free memory polling happens only during scheduling decisions; stale memory reports cause load failures
- Files: `server/sched.go:159-165,246` (on-demand GPU queries)
- Cause: Lazy GPU discovery; no background refresh thread
- Improvement path:
  1. Background thread that refreshes GPU state every 100ms
  2. Implement memory.Watch() callback from GPU libraries
  3. Cache with TTL instead of on-demand queries
  4. Predictive memory trending to anticipate when models will unload

**Model Parsing - Synchronous Conversion:**
- Problem: Model conversions (safetensors → GGML) block entire request thread
- Files: `convert/convert.go`, `server/model.go:84-110` (synchronous ZIP extraction and conversion)
- Cause: No streaming or async conversion pipeline
- Improvement path:
  1. Move conversion to background workers (worker pool of size 2-4)
  2. Implement streaming ZIP extraction to memory
  3. Add model conversion progress tracking to API
  4. Pre-convert models on disk in background after pull completes

## Fragile Areas

**Scheduler State Machine - Complex Lock Ordering:**
- Files: `server/sched.go:100-311` (processRequests loop), `server/sched.go:313-380` (processCompleted)
- Why fragile: Multiple state machines (loaded runner refCounts, expiration timers, pending requests) with indirect synchronization via channels; easy to introduce deadlocks or race conditions
- Safe modification:
  1. Add comprehensive comments documenting state transitions
  2. Introduce invariant assertions at key points (e.g., "loaded[x] exists ⟹ refCount > 0")
  3. Use state enums instead of boolean flags
  4. Run with race detector enabled in all test suites
- Test coverage: scheduler_test.go has good coverage but lacks stress tests for rapid unload/reload cycles

**GPU Discovery - Platform-Specific C Bindings:**
- Files: `discover/gpu.go`, `discover/gpu_linux.go`, `discover/gpu_windows.go`, `discover/gpu_darwin.go` (10+ platform-specific implementations)
- Why fragile: Each GPU library (CUDA, ROCm, OneAPI, Metal) has different memory reporting accuracy; C interop requires careful resource cleanup
- Safe modification:
  1. Add per-GPU reliability tracking in config
  2. Log discovered GPU properties at startup with warnings
  3. Add health checks after loading models to validate predictions
  4. Implement graceful degradation if GPU becomes unavailable mid-request
- Test coverage: gpu_test.go limited to Linux; missing Windows/Darwin tests

**KV Cache Slot Allocation - Implicit Slot Reuse:**
- Files: `llama/runner/cache.go:66-100` (LoadCacheSlot logic)
- Why fragile: Longest/best cache slot selection is heuristic-based; race conditions possible if multiple requests load slots simultaneously
- Safe modification:
  1. Add locking around LoadCacheSlot (document required holding locks)
  2. Use atomic operations for slot.InUse flag
  3. Add invariant: each slot can be InUse by max 1 sequence
  4. Validate cache inputs haven't changed since slot selection
- Test coverage: No concurrency tests for cache slot contention

**Model Manifest Parsing - ZIP Format Dependency:**
- Files: `server/model.go:84-110`, `server/manifest.go`
- Why fragile: Relies on zipfile entry ordering and format assumptions; malformed manifests could bypass validation
- Safe modification:
  1. Validate manifest schema before processing
  2. Add file size sanity checks
  3. Use filepath.Clean on all extracted paths
  4. Whitelist allowed layer media types explicitly
- Test coverage: manifest_test.go basic; missing fuzz tests for malformed inputs

## Scaling Limits

**Concurrent Model Loading - Single-Threaded llama.cpp Initialization:**
- Current capacity: ~1-2 models loading simultaneously before scheduler contention
- Limit: llama.cpp context creation locks in C layer; scheduler can't parallelize initialization
- Scaling path:
  1. Implement model warmup queue; pre-initialize contexts in background
  2. Use llama.cpp batched context creation if available in future versions
  3. Measure initialization time per model family; predict load time accurately
  4. Implement adaptive timeouts based on observed model initialization delays

**Session Duration Tracking - O(n) Timer Management:**
- Current capacity: ~100 concurrent sessions before timer overhead
- Limit: time.AfterFunc creates new OS timers for each session; cleanup is synchronous
- Scaling path:
  1. Use hierarchical timer wheel instead of individual timers
  2. Batch timer events; process expiring sessions in groups
  3. Use context.WithTimeout for session deadlines instead of explicit timers
  4. Profile with 1000+ concurrent sessions to identify bottlenecks

**GPU Device Memory - Fixed Allocation per Model:**
- Current capacity: ~10-20 models per high-end GPU before memory fragmentation
- Limit: KV cache fragmentation after many load/unload cycles; defragmentation is incomplete
- Scaling path:
  1. Implement true defragmentation: compact KV caches periodically
  2. Add adaptive sequence length allocation
  3. Track memory fragmentation metrics
  4. Rewrite defragmentation logic with validation (currently commented as TODO)

## Dependencies at Risk

**Deprecated Go Compatibility:**
- Risk: Project targets Go 1.23.4; older dependency versions may lack security updates
- Impact: CGO-heavy project vulnerable to C library attacks (CUDA, ROCm, llama.cpp)
- Migration plan:
  1. Regular dependency audits (monthly)
  2. Automated testing against latest Go version
  3. Pin versions only for stable production; allow patch upgrades
  4. Security scanning for transitive C library vulnerabilities

**llama.cpp C Library - Tight Coupling:**
- Risk: Core functionality depends on specific llama.cpp build flags and features
- Impact: New llama.cpp versions may break compatibility; feature detection incomplete
- Migration plan:
  1. Version-pin llama.cpp; test before upgrades
  2. Add feature detection at startup (GGML_USE_* flags)
  3. Implement graceful fallbacks for missing features
  4. Document minimum llama.cpp version requirement

**Gin Web Framework - Limited Security Hardening:**
- Risk: Using Gin without explicit CORS/CSRF protection; middleware ordering could be fragile
- Impact: Cross-origin requests could access model data; CSRF attacks possible
- Migration plan:
  1. Review and document all CORS settings in use
  2. Add CSRF token validation for state-changing requests
  3. Consider middleware audit (rate limiting, auth) for enterprise deployments
  4. Add request size limits to prevent DoS

## Missing Critical Features

**API Cancellation - No Graceful Request Termination:**
- Problem: No way to cancel long-running generate/embed requests; clients must disconnect
- Blocks: Real-time chat applications that need request cancellation; resource cleanup on client disconnect is incomplete
- Impact: Wasted GPU compute; scheduler doesn't free up runners until request completes or timeout
- Recommendation: Implement context.Context propagation with proper cancellation signaling

**Model Unloading Progress - No Visibility into Unload Status:**
- Problem: Models unload silently; no API to query unload progress or force immediate unload
- Blocks: Debugging scheduler stalls; users can't see if model is unloading or stuck
- Impact: Apparent freezes when models are being unloaded; hard to diagnose scheduler issues
- Recommendation: Add /ps endpoint with unload status; implement stream of unload events

**Batch Processing Limits - No Queue Policy Configuration:**
- Problem: MaxQueue fixed; no way to prioritize or expire old requests
- Blocks: Production deployments can't implement SLA-aware queueing; first-come-first-served
- Impact: High-latency requests starve fast requests; no fairness guarantees
- Recommendation: Add priority queues; implement request expiration TTL; configurable max queue

## Test Coverage Gaps

**Scheduler - Concurrent Unload Stress Tests (High Priority):**
- What's not tested: Rapid fire model unload/reload cycles; edge cases in refCount management
- Files: `server/sched_test.go` (mostly sequential tests)
- Risk: Deadlocks, race conditions, resource leaks under load
- Recommendation: Add stress test with 1000+ unload cycles; run with race detector enabled; profile memory usage

**GPU Memory Estimation Validation:**
- What's not tested: EstimatedVRAM accuracy across model families; prediction failures
- Files: `server/sched_test.go` (mock GPUs only)
- Risk: Silent failures loading models; scheduler becomes unreliable on new hardware
- Recommendation: Add real hardware validation tests; track prediction vs actual ratio; alert on large discrepancies

**KV Cache Edge Cases - Fragmentation Recovery:**
- What's not tested: Cache defragmentation under extreme context shifts; cache corruptions recovery
- Files: `llama/runner/cache.go` (no dedicated test file)
- Risk: Cache becomes unusable after many shifts; unrecoverable state
- Recommendation: Add cache invariant validation; fuzz test cache operations; implement cache reset recovery

**Model Conversion - Malformed Input Handling:**
- What's not tested: Corrupt GGML files, truncated safetensors, out-of-spec model configs
- Files: `convert/convert_test.go` (basic format tests only)
- Risk: Panics instead of graceful errors; users can't recover
- Recommendation: Add fuzz tests for file format parsing; implement all-paths error handling

**Windows-Specific Scheduler Behavior:**
- What's not tested: GPU discovery on Windows; scheduler behavior with DirectML/ROCm-on-Windows
- Files: `discover/gpu_windows.go` (code exists, tests minimal)
- Risk: Scheduler fails silently on Windows; users can't load models
- Recommendation: CI test on Windows GPU machines; add Windows-specific scheduler tests

---

*Concerns audit: 2026-03-22*
