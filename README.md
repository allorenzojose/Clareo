 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/README.md b/README.md
index dd288fef34b40665d7c8d482903af7037789eca2..c250e8a092f2dd34cba9d522c4e34b109d2c9803 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,10 @@
-git add .
-git commit -m "Initial commit"
-git push origin main
+# Clareo – AI Exposure Optimization SaaS
+
+This repository contains the master blueprint for building Clareo, a SaaS platform that measures and improves how brands appear inside LLMs such as ChatGPT, Claude, Gemini, and Copilot. The documentation covers:
+
+- Product goals, ICP, and module definitions
+- System architecture, database schema, and backend workflows
+- API specs, prompt chains, dashboards, and UI flows
+- Roadmap, pricing, user stories, and engineering backlog
+
+👉 Read the full specification in [`docs/aeo-llm-visibility-saas.md`](docs/aeo-llm-visibility-saas.md).
 
EOF
)
