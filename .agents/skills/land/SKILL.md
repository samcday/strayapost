---
name: land
description: Validate and publish OpenSpec changes as accepted specifications. Use for /land and Delta's Land Changes action.
metadata:
  delta-action: land
---

# Land specifications

This project ships specifications, not their implementation.
Do not implement the described software or mark implementation tasks complete.

Install the locked tooling with `npm ci --ignore-scripts`.
Use `npx --no-install openspec` for OpenSpec.
Load `report-landed` before carrying out the landing.

1. Identify the change and authorized destination.
   Read the project instructions and OpenSpec configuration.
   Inspect the remote, preserve unrelated work, and use a private working branch.

2. Review the change and affected main specs.
   Check that requirements are clear, scenarios establish observable outcomes,
   and the artifacts agree. Check compatibility and supporting evidence.
   Run `npm run check`; syntax checks alone are not enough.

3. Rebase onto the latest destination.
   Resolve mechanical conflicts while preserving independent requirements and
   scenarios. Ask when changes express competing behavior.
   Repeat review and validation after integration.

4. Use OpenSpec's archive workflow to apply the deltas to the main specs
   and archive the change.
   Run `npm run check` again and inspect the resulting specs and final diff.
   Check for lost scenarios, contradictory requirements, and unrelated changes.
   Never weaken a requirement merely to make validation pass.

5. Commit the agreed changes and require passing CI for the candidate commit
   before updating the destination. Publish without force-pushing.
   If the destination advances, integrate it and repeat the checks.
   Verify that the commit reached the remote destination before reporting success.
   Report the checks performed and their limits using `report-landed`.
   After a verified landing, message the parent thread with the remote branch
   and full commit SHA. Ask it to rebase onto that SHA while preserving local work.
   If there is no parent thread, omit that message.
