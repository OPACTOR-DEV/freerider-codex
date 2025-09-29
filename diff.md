diff --git a/codex-rs/core/src/config.rs b/codex-rs/core/src/config.rs
index 292b9f7b..33851199 100644
--- a/codex-rs/core/src/config.rs
+++ b/codex-rs/core/src/config.rs
@@ -170,6 +170,9 @@ pub struct Config {
     /// Base URL for requests to ChatGPT (as opposed to the OpenAI API).
     pub chatgpt_base_url: String,
 
+    /// Experimental rollout resume path (absolute or relative to the configured cwd).
+    pub experimental_resume: Option<PathBuf>,
+
     /// Include an experimental plan tool that the model can use to update its current plan and status of each step.
     pub include_plan_tool: bool,
 
@@ -703,6 +706,9 @@ pub struct ConfigToml {
     /// Base URL for requests to ChatGPT (as opposed to the OpenAI API).
     pub chatgpt_base_url: Option<String>,
 
+    /// Experimental path to a rollout JSONL used to resume a session.
+    pub experimental_resume: Option<PathBuf>,
+
     /// Experimental path to a file whose contents replace the built-in BASE_INSTRUCTIONS.
     pub experimental_instructions_file: Option<PathBuf>,
 
@@ -952,6 +958,14 @@ impl Config {
 
         let history = cfg.history.unwrap_or_default();
 
+        let experimental_resume = cfg.experimental_resume.map(|path| {
+            if path.is_absolute() {
+                path
+            } else {
+                resolved_cwd.join(path)
+            }
+        });
+
         let tools_web_search_request = override_tools_web_search_request
             .or(cfg.tools.as_ref().and_then(|t| t.web_search))
             .unwrap_or(false);
@@ -1050,6 +1064,7 @@ impl Config {
                 .chatgpt_base_url
                 .or(cfg.chatgpt_base_url)
                 .unwrap_or("https://chatgpt.com/backend-api/".to_string()),
+            experimental_resume,
             include_plan_tool: include_plan_tool.unwrap_or(false),
             include_apply_patch_tool: include_apply_patch_tool.unwrap_or(false),
             tools_web_search_request,
@@ -1798,6 +1813,7 @@ model_verbosity = "high"
                 model_reasoning_summary: ReasoningSummary::Detailed,
                 model_verbosity: None,
                 chatgpt_base_url: "https://chatgpt.com/backend-api/".to_string(),
+                experimental_resume: None,
                 base_instructions: None,
                 include_plan_tool: false,
                 include_apply_patch_tool: false,
@@ -1857,6 +1873,7 @@ model_verbosity = "high"
             model_reasoning_summary: ReasoningSummary::default(),
             model_verbosity: None,
             chatgpt_base_url: "https://chatgpt.com/backend-api/".to_string(),
+            experimental_resume: None,
             base_instructions: None,
             include_plan_tool: false,
             include_apply_patch_tool: false,
@@ -1931,6 +1948,7 @@ model_verbosity = "high"
             model_reasoning_summary: ReasoningSummary::default(),
             model_verbosity: None,
             chatgpt_base_url: "https://chatgpt.com/backend-api/".to_string(),
+            experimental_resume: None,
             base_instructions: None,
             include_plan_tool: false,
             include_apply_patch_tool: false,
@@ -1991,6 +2009,7 @@ model_verbosity = "high"
             model_reasoning_summary: ReasoningSummary::Detailed,
             model_verbosity: Some(Verbosity::High),
             chatgpt_base_url: "https://chatgpt.com/backend-api/".to_string(),
+            experimental_resume: None,
             base_instructions: None,
             include_plan_tool: false,
             include_apply_patch_tool: false,
diff --git a/codex-rs/core/src/conversation_manager.rs b/codex-rs/core/src/conversation_manager.rs
index 51854248..10f7c1f1 100644
--- a/codex-rs/core/src/conversation_manager.rs
+++ b/codex-rs/core/src/conversation_manager.rs
@@ -61,11 +61,20 @@ impl ConversationManager {
         config: Config,
         auth_manager: Arc<AuthManager>,
     ) -> CodexResult<NewConversation> {
-        let CodexSpawnOk {
-            codex,
-            conversation_id,
-        } = Codex::spawn(config, auth_manager, InitialHistory::New).await?;
-        self.finalize_spawn(codex, conversation_id).await
+        if let Some(resume_path) = config.experimental_resume.clone() {
+            let initial_history = RolloutRecorder::get_rollout_history(&resume_path).await?;
+            let CodexSpawnOk {
+                codex,
+                conversation_id,
+            } = Codex::spawn(config, auth_manager, initial_history).await?;
+            self.finalize_spawn(codex, conversation_id).await
+        } else {
+            let CodexSpawnOk {
+                codex,
+                conversation_id,
+            } = Codex::spawn(config, auth_manager, InitialHistory::New).await?;
+            self.finalize_spawn(codex, conversation_id).await
+        }
     }
 
     async fn finalize_spawn(
diff --git a/docs/config.md b/docs/config.md
index ba204ee0..e6919715 100644
--- a/docs/config.md
+++ b/docs/config.md
@@ -650,7 +650,7 @@ notifications = [ "agent-turn-complete", "approval-requested" ]
 | `model_supports_reasoning_summaries` | boolean | Force‑enable reasoning summaries. |
 | `model_reasoning_summary_format` | `none` \| `experimental` | Force reasoning summary format. |
 | `chatgpt_base_url` | string | Base URL for ChatGPT auth flow. |
-| `experimental_resume` | string (path) | Resume JSONL path (internal/experimental). |
+| `experimental_resume` | string (path) | Resume JSONL path (internal/experimental). Relative paths are resolved against the configured working directory. |
 | `experimental_instructions_file` | string (path) | Replace built‑in instructions (experimental). |
 | `experimental_use_exec_command_tool` | boolean | Use experimental exec command tool. |
 | `responses_originator_header_internal_override` | string | Override `originator` header value. |
