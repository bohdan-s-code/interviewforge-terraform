# Terraform Learning Plan: InterviewForge

This plan teaches Terraform through one pet project, **InterviewForge**, a tool that helps a technical interviewer prepare interview plans and feedback.

Each feature in the project exists to teach a specific part of Terraform. All technical decisions (stack, architecture, structure) are left for implementation planning.

---

## Features list

Each feature is listed with the Terraform topic it is there to teach.

1. **Interview plans** (create and view plans by role, stack, and seniority). Teaches: basic resources, variables, outputs, passing infrastructure values to the app.
2. **Transcript upload**. Teaches: storage resources, access policies, encryption, CORS.
3. **Feedback generation, run asynchronously**. Teaches: connecting services to each other, triggers, permissions between components.
4. **Feedback history per candidate**. Teaches: data resources, protecting stateful resources from deletion.
5. **Feedback export**. Teaches: reusing one module for a second, similar resource.
6. **Single-user login**. Teaches: secrets, sensitive values, keeping secrets out of state.
7. **Automatic cleanup of old transcripts**. Teaches: scheduled resources, lifecycle rules.
8. **Web UI**. Teaches: static hosting, multiple providers in one configuration.
9. **Separate dev and prod environments**. Teaches: environment separation, remote state, CI/CD for infrastructure.
10. **Monitoring and alerts**. Teaches: observability resources and alarms defined in code.

---

## Phases

### Phase 0: Orientation

**Learn**
1. What Infrastructure as Code is, and the difference between declarative and imperative.
2. The core Terraform workflow: `init`, `plan`, `apply`, `destroy`.
3. What providers are and how the registry works.
4. What the lock file and the state file are.
5. Terraform vs OpenTofu.

**Implement**
1. Install Terraform and the supporting tools.
2. Create a first throwaway configuration using local-only providers.
3. Run the full workflow end to end and read every line of the plan output.
4. Edit a managed file by hand and observe drift for the first time.

**Done when:** you can explain what each command does and what each generated file is for.

---

### Phase 1: Core language, run locally at no cost

**Learn**
1. Resources and data sources.
2. Dependencies, both implicit and explicit (`depends_on`).
3. Input variables: types, defaults, validation, sensitive values.
4. Outputs and locals.
5. Expressions: string templates, conditionals, `for` expressions.
6. `count` vs `for_each`.
7. Common built-in functions.
8. The difference between "update in place" and "replace" in a plan.

**Implement**
1. Feature 1, Interview plans: run it locally with Terraform managing every piece the app needs.
2. Feature 2, Transcript upload: use local storage first.
3. Feature 3, Feedback generation: use a stubbed result, with no real AI calls.
4. Pass connection details from Terraform outputs into the app.

**Break and fix**
1. Remove a managed component by hand, then let Terraform recreate it.
2. Change a version or setting that forces a replacement, and explain why it does.
3. Switch from `count` to `for_each` and observe how resource addresses change.

**Done when:** a single `apply` from a clean machine gives you a working local app.

---

### Phase 2: State, refactoring, modules

**Learn**
1. What state contains and why it matters, including secrets stored in plaintext.
2. State commands: `list`, `show`, `mv`, `rm`.
3. Importing existing resources (`import` blocks).
4. Safe refactoring with `moved` blocks.
5. Removing resources from management with `removed` blocks.
6. Modules: inputs, outputs, composition.
7. `terraform console` for experimenting with expressions.

**Implement**
1. Extract the repeated parts of the configuration into reusable modules.
2. Refactor without recreating anything, using `moved`.
3. Bring a manually created resource under Terraform management.
4. Add validation rules to module inputs.
5. Generate documentation for your modules.

**Break and fix**
1. Do a refactor without `moved` first, read the destructive plan, then fix it with `moved`.
2. Remove a resource from state and re-import it.
3. Find a secret in the state file and note it as a lesson.

**Done when:** you can restructure your configuration freely without recreating running infrastructure.

---

### Phase 3: Quality gates and testing

**Learn**
1. Formatting and validation.
2. Linting and security scanning for Terraform code.
3. `terraform test`: test files, `run` blocks, assertions.
4. Mocked providers, so tests need no infrastructure.
5. Preconditions, postconditions, `check` blocks.
6. How variable files work and the order in which variable values take precedence.

**Implement**
1. Automatic format, validate, lint, and scan checks on every commit.
2. Tests for your modules, with at least one that uses a mocked provider.
3. A precondition or postcondition that catches a misconfiguration.
4. A CI pipeline that runs every check on each pull request.

**Break and fix**
1. Make a test fail on purpose and read the output.
2. Push unformatted code and confirm CI blocks it.

**Done when:** every change gets automated feedback before it is merged.

---

### Phase 4: Cloud concepts on a local cloud emulator

**Learn**
1. Cloud provider configuration: regions, default tags, custom endpoints.
2. Identity and permissions: roles, policies, least privilege.
3. Object storage: access blocking, versioning, encryption, lifecycle rules.
4. Queues and dead-letter queues.
5. Serverless functions and event triggers.
6. Managed secrets.
7. Multiple provider configurations and aliases.
8. `lifecycle` options: `prevent_destroy`, `create_before_destroy`, `ignore_changes`.

**Implement**
1. Move Feature 2, Transcript upload, to cloud-style storage.
2. Move Feature 3, Feedback generation, to a queue-driven asynchronous flow with failure handling.
3. Feature 4, Feedback history, with deletion protection.
4. Feature 5, Feedback export, reusing your storage module.
5. Feature 6, Single-user login, with a managed secret readable only by the component that needs it.
6. Feature 7, Cleanup, as a lifecycle rule.
7. An end-to-end smoke check that uses Terraform outputs.

**Break and fix**
1. Remove a permission and trace the failure.
2. Make the async processing fail and confirm failed messages are captured.
3. Rename a resource that forces replacement, then protect it.

**Done when:** the full flow (plan → upload → feedback → export) works against the emulator.

---

### Phase 5: Real cloud on a free plan

**Learn**
1. Setting up a free cloud account safely: MFA, a non-root user, budget alerts.
2. Remote state with locking.
3. The bootstrap problem: who creates the state storage.
4. Migrating state from local to remote.
5. Sharing data between configurations.
6. CI authentication without long-lived keys (OIDC).
7. The workflow of running `plan` on a pull request and `apply` on merge.
8. Naming and tagging conventions.
9. Handling secrets so they stay out of state.

**Implement**
1. A one-time bootstrap configuration for remote state and CI access.
2. Migrate to remote state.
3. Deploy everything from Phase 4 to the real cloud.
4. Feature 8, Web UI, with static hosting. Optionally use a second provider for it.
5. Feature 7, Cleanup, adding a scheduled job.
6. Feature 9, Environments: dev deploys automatically, prod deploys after manual approval.

**Break and fix**
1. Change something in the cloud console and detect the drift.
2. Run two applies at the same time and watch locking work.
3. Roll back a bad change through Git.
4. Destroy dev completely and rebuild it from scratch.

**Done when:** a merged PR deploys dev, prod deploys with approval, and a full destroy-and-rebuild works.

---

### Phase 6: Production hardening

**Learn**
1. Fixing findings from security scans.
2. Module versioning and pinning.
3. Reading well-known community modules.
4. Observability as code.
5. Cost awareness.
6. Policy as code basics.

**Implement**
1. Fix or justify every high-severity scan finding.
2. Version your modules and pin each environment to a version.
3. Feature 10, Monitoring and alerts.
4. One policy-as-code rule checked against the plan.
5. A runbook covering bootstrap, deploy, rollback, destroy, and secret rotation.

**Done when:** another engineer could deploy their own copy using only the runbook.

---

### Phase 7 (optional): Make it real

**Learn**
1. Handling real personal data: encryption, retention, access logging.

**Implement**
1. Replace the stubbed feedback with a real AI provider.
2. Add real interview plan templates.
3. Decide whether the tool will ever hold real candidate data. If it will, harden it first.
4. Write up what you learned as an article.

---

## Ground rules

1. Use synthetic data only while learning.
2. Make changes only through Terraform. Use the cloud console only to look.
3. Destroy paid-when-idle resources at the end of each session.
4. Commit after every exercise.
5. Keep the app minimal, because the goal is Terraform.
6. Split each session into three parts: about 20% learning the concept, 50% building, and 30% breaking and fixing.

---

## Suggested timeline

1. Week 0: Phase 0.
2. Week 1: Phase 1.
3. Week 2: Phase 2.
4. Week 3: Phase 3.
5. Weeks 4–5: Phase 4.
6. Weeks 6–7: Phase 5.
7. Week 8: Phase 6.
8. Week 9 and later: Phase 7.
