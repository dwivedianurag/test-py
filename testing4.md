Issue: A1 artifact ingestion + semantic search inconsistencies

Summary
- Artifact upload calls return validation errors (unknown op) when invoked via tools/call; direct A1 JSON-RPC works.
- A1 docs ingest succeeds (including artifact-based ingest), but doc keyword search returns reason:"no_embeddings" even when docs exist; SHA-exact lookup works.
- Duplicate key errors appear in semantic embedding logs when re-ingesting the same doc_id.

Environment
- A1 HTTP MCP at http://127.0.0.1:8080/mcp.
- Tools guide: A1-tools-guide-YAML-v2.md (v2.1).

What we attempted (chronological)
1) A1 artifact uploads via tools/call (failed)
   - Calls: a1.artifacts.ingest_bytes and a1.artifacts.upload.begin/part/finish via tools/call.
   - Errors:
     - VALIDATION_ERROR for artifacts calls (invalid request).
     - a1_client_error op=a1_artifacts_ingest_bytes code=-32602 msg=unknown op a1_artifacts_ingest_bytes
     - a1_client_error op=a1_artifacts_upload_begin code=-32602 msg=unknown op a1_artifacts_upload_begin
     - a1_client_error op=tools/call code=-32602 msg=unknown op tools/call
   - Workaround: call A1 directly via HTTP JSON-RPC method names (a1.artifacts.ingest_bytes, etc.).

2) A1 ingest via inline content (worked)
   - Cloned repo: https://github.com/dwivedianurag/test-py.git
   - Files found: app.py, test.py, README.md.
   - Docs ingest: a1.docs.ingest_uris with content + labels {tag:"test2"}.
     - Job: job-a015384021f64073b4cd762684304869
   - Code ingest: a1.code.ingest_repo with content for each file (mode: activate).
     - Job: job-3af29980fcf340feaef709d94fcc8f65
   - Verified symbol present: server_connector_hub.app.hello_Qstar.
   - Verified doc present via SHA-exact lookup: README.md.

3) A1 ingest via artifacts.ingest_bytes (worked)
   - Used A1 artifact ingest for each file, then a1.docs.ingest_uris and a1.code.ingest_repo with artifact_id.
   - Jobs:
     - Docs: job-ad1cd61c9748404ca05f8af2bff56a4d
     - Code: job-e757b5577ea3470698c413267b42969e
   - Artifact ids:
     - Doc: artifact-d76fbdffe07e44d49e650cf7c9682413
     - Code: artifact-7f91e3f86ceb4c72a905dbd1bb533700, artifact-f6cc8b779bdd43649ebfb66e27116d4a

4) Re-ingest via artifacts (worked, with doc skip to avoid duplicates)
   - Re-ingested the repo; skipped README.md if SHA already present.
   - Jobs:
     - Docs: job-27759667a1884e598063e7eb4c3d71c8
     - Code: job-4c2ba510293b4121a1c35ce82e732b88
   - Code artifacts: artifact-a0b422b31b9d4ee5849f40756e66d3ce, artifact-3cec1b9b8bf347998a8b5c9bf48f4386, artifact-5e4816e361bb4d41b426ae4556ec70cd

5) Ingest remote docs via artifact ingest
   - Testing.md:
     - Raw URL: https://raw.githubusercontent.com/dwivedianurag/test-py/main/Testing.md
     - SHA256: f657d548071353afd8501a112d8f3b57ce1f5c7a349486cd935fd68be02e2ca9
     - Artifact: artifact-e307ca60d2d7462c8b7a586e567de7be
     - Docs job: job-a466933539484bf4ab10ec390492a06c
   - testcon.md:
     - Raw URL: https://raw.githubusercontent.com/dwivedianurag/test-py/main/testcon.md
     - SHA256: 1fd2141ecb1a3262300fe4f145a6f720c998151c2e97c5f3f43714306e86d7ca
     - Artifact: artifact-7436228267454ea58bbf9bdd5586d6f9
     - Docs job: job-6b3d3c1fdca14d01ba3a689176a8471f

Observed issues
1) A1 tools/call rejects artifact ops
   - Error: "unknown op tools/call" when invoking tools/call against A1.
   - Error: "unknown op a1_artifacts_ingest_bytes" and "unknown op a1_artifacts_upload_begin".
   - Workaround: call direct JSON-RPC methods (a1.artifacts.ingest_bytes, etc.).

2) Semantic search returns no results even when docs exist
   - a1.search(scope:"docs", q:"Azure"/"maximum"/"Testing.md"/"massive") -> items:[] with reason:"no_embeddings".
   - SHA-exact lookup works (doc is present).
   - This persists even after artifact-based ingest of new docs (Testing.md, testcon.md).
   - Log error:
     - Duplicate key: doc_id + chunk_id already exists.
   - Hypothesis: re-ingest or duplicate doc_ids prevent embedding writes; semantic coverage gating keeps search empty.

3) Artifact API response shape differs between tools/call and direct op
   - tools/call returns result.content[0].text (envelope).
   - direct op returns {result:{...}, receipts:{...}}.
   - Needed parser for both shapes to retrieve artifact_id.

Open questions / help needed
1) A1 bridge compatibility:
   - Does the A1 MCP deployment intentionally disable tools/call for artifact ops?
   - Is there a supported bridge or transport variant that fully exposes tools/call?

2) Semantic embeddings:
   - How should duplicates be handled (replace or skip) to avoid PK collisions?
   - Is semantic readiness gated by A1_SEMANTIC_COVERAGE_MIN, and can we lower it for testing?
   - How to verify doc semantic embeddings exist (admin.verify fields and thresholds)?

Desired outcome
- A1 tools/call supports artifact ops (or a documented supported transport is provided).
- Doc keyword search (a1.search scope:"docs") returns results for newly ingested docs without requiring SHA-exact lookup.
