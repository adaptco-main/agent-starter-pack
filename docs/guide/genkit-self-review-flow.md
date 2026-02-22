# Genkit self-review flow pattern (TypeScript)

If you want to adapt a Genkit app to run a **generate → apply tool → self-review** loop, use this pattern.

> Note: The current `agent-starter-pack` repository does not contain a backend `index.ts` + `agent.ts` Genkit entrypoint, so this is provided as an integration reference for projects that do.

## Recommended implementation

```ts
import { configureGenkit, defineFlow } from 'genkit';
import { vertexAI } from 'genkitx-vertexai';
import { z } from 'zod';

import { writeToFile } from './agent';

configureGenkit({
  plugins: [vertexAI()],
  logLevel: 'debug',
});

export const agentFlow = defineFlow(
  {
    name: 'agentFlow',
    inputSchema: z.string(),
    outputSchema: z.object({
      status: z.string(),
      analysis: z.string().optional(),
    }),
  },
  async (prompt) => {
    const model = vertexAI.model('gemini-1.5-flash-preview');

    // 1) Primary generation with tool availability.
    const initial = await model.generate({
      prompt,
      tools: [writeToFile],
    });

    // 2) If a tool call is requested, execute it, then run a verification pass.
    const toolRequest = initial.toolRequest();
    if (toolRequest) {
      await toolRequest.run();

      const targetPath =
        typeof toolRequest.params?.path === 'string'
          ? toolRequest.params.path
          : 'the generated artifact';

      const review = await model.generate({
        prompt: [
          `Review the deployment/update of ${targetPath}.`,
          'Confirm the code addresses previous failure patterns found in firestore_export logs.',
          'Return a concise risk summary and any required follow-up actions.',
        ].join(' '),
      });

      return { status: 'Deployed & Verified', analysis: review.text() };
    }

    return { status: 'Completed', analysis: initial.text() };
  }
);
```

## Why this shape

- Keeps tool execution explicit and auditable.
- Uses a second LLM pass only when side effects occur.
- Gracefully handles missing `toolRequest.params.path`.
- Produces a stable output contract (`status`, optional `analysis`) for orchestration layers.

## Optional hardening

- Add a timeout/retry strategy around `toolRequest.run()`.
- Add path allow-listing before file writes.
- Include structured review output (JSON schema) rather than free text.
