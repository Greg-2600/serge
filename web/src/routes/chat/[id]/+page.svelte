<script lang="ts">
  import type { PageData } from "./$types";
  import { invalidate, goto } from "$app/navigation";
  import { page } from "$app/stores";

  import { onMount } from "svelte";
  import ClipboardJS from "clipboard";
  import hljs from "highlight.js";
  import "highlight.js/styles/github-dark.css";
  import javascript from "highlight.js/lib/languages/javascript";
  import typescript from "highlight.js/lib/languages/typescript";
  import rust from "highlight.js/lib/languages/rust";
  import python from "highlight.js/lib/languages/python";
  import MarkdownIt from "markdown-it";
  import mdHighlight from "markdown-it-highlightjs";

  hljs.registerLanguage("javascript", javascript);
  hljs.registerLanguage("typescript", typescript);
  hljs.registerLanguage("rust", rust);
  hljs.registerLanguage("python", python);

  export let data: PageData;

  const isLoading = false;
  $: startDate = new Date(data.chat.created);
  $: history = data.chat.history;

  let prompt = "";

  async function askQuestion() {
    const data = new URLSearchParams();

    if (!prompt || prompt === "") {
      prompt = "Reformulate your last answer.";
    }

    data.append("prompt", prompt);

    const eventSource = new EventSource(
      "/api/chat/" + $page.params.id + "/question?" + data.toString(),
    );

    history = [
      ...history,
      {
        type: "human",
        data: {
          content: prompt,
        },
      },
      {
        type: "ai",
        data: {
          content: "",
        },
      },
    ];

    prompt = "";

    eventSource.addEventListener("message", (event) => {
      history[history.length - 1].data.content += event.data;
    });

    eventSource.addEventListener("close", async () => {
      eventSource.close();
      await invalidate("/api/chat/" + $page.params.id);
    });

    eventSource.onerror = async (error) => {
      console.log("error", error);
      eventSource.close();
      //history[history.length - 1].data.content = "A server error occurred.";
      //await invalidate("/api/chat/" + $page.params.id);
    };
  }

  async function handleKeyDown(event: KeyboardEvent) {
    if (event.key === "Enter" && event.ctrlKey) {
      await askQuestion();
    }
  }

  async function createSameSession() {
    const newData = await fetch(
      `/api/chat/?model=${data.chat.params.model_path}&temperature=${data.chat.params.temperature}&top_k=${data.chat.params.top_k}` +
        `&top_p=${data.chat.params.top_p}&max_length=${data.chat.params.max_tokens}&context_window=${data.chat.params.n_ctx}` +
        `&repeat_last_n=${data.chat.params.last_n_tokens_size}&repeat_penalty=${data.chat.params.repeat_penalty}` +
        `&n_threads=${data.chat.params.n_threads}&init_prompt=${data.chat.history[0].data.content}` +
        `&gpu_layers=${data.chat.params.n_gpu_layers}&n_gqa=${data.chat.params.n_gqa}`,

      {
        method: "POST",
        headers: {
          accept: "application/json",
        },
      },
    ).then((response) => response.json());
    await invalidate("/api/chat/");
    await goto("/chat/" + newData);
  }

  document.addEventListener("keydown", async (event) => {
    if (event.key === "n" && event.altKey) {
      await createSameSession();
    }
  });

  async function deletePrompt(chatID: string, idx: number) {
    const response = await fetch(
      `/api/chat/${chatID}/prompt?idx=${idx.toString()}`,
      { method: "DELETE" },
    );

    if (response.status === 200) {
      await invalidate("/api/chat/" + $page.params.id);
    } else {
      console.error("Error " + response.status + ": " + response.statusText);
    }
  }

  const md: MarkdownIt = new MarkdownIt({
    html: true,
    linkify: true,
    typographer: true,
    breaks: true,
    highlight: (code_string: string, lang: string) => {
      if (lang && hljs.getLanguage(lang)) {
        try {
          const code = hljs.highlight(code_string, lang).value;
          return hljs.highlight(code, { language: lang }).value;
        } catch (ex) {
          /**/
        }
      }
      return "";
    },
  }).use(mdHighlight, { hljs });

  const originalFenceRenderer = md.renderer.rules.fence;

  md.renderer.rules.fence = (
    tokens: any,
    index: any,
    options: any,
    env: any,
    self: any,
  ) => {
    // Increment the codeblock id
    const id = `code-block-${Math.random().toString(36).substring(7)}`;
    // Generate original fenced code block HTML
    if (!originalFenceRenderer)
      throw new Error("originalFenceRenderer is undefined");

    const codeBlock = originalFenceRenderer(
      tokens,
      index,
      options,
      env,
      self,
    ).replace("<code", `<code id="code-block-${id}"`);

    // Get the token and code content
    const token = tokens[index];
    const content = token.content;

    // Create copy button HTML
    const copyButton = `<button class="copy-button cursor-pointer color-base-content bg-base-300 py-0.5 px-2 mt-1 mr-1 border rounded border-base-content absolute top-0 right-1 opacity-20"  data-clipboard-target="#code-block-${id}"><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" width="12" height="12"><path class="fill-base-content" d="M0 6.75C0 5.784.784 5 1.75 5h1.5a.75.75 0 0 1 0 1.5h-1.5a.25.25 0 0 0-.25.25v7.5c0 .138.112.25.25.25h7.5a.25.25 0 0 0 .25-.25v-1.5a.75.75 0 0 1 1.5 0v1.5A1.75 1.75 0 0 1 9.25 16h-7.5A1.75 1.75 0 0 1 0 14.25Z"></path><path class="fill-base-content" d="M5 1.75C5 .784 5.784 0 6.75 0h7.5C15.216 0 16 .784 16 1.75v7.5A1.75 1.75 0 0 1 14.25 11h-7.5A1.75 1.75 0 0 1 5 9.25Zm1.75-.25a.25.25 0 0 0-.25.25v7.5c0 .138.112.25.25.25h7.5a.25.25 0 0 0 .25-.25v-7.5a.25.25 0 0 0-.25-.25Z"></path></svg></button>`;

    // Wrap the fenced code block and copy button in a container
    const html = `
      <div class="my-2.5 w-full relative overflow-x-hidden">
        ${copyButton}
        ${codeBlock}
      </div>
    `;

    return html;
  };

  const renderMarkdown = (q: any) => {
    return md.render(q);
  };

  onMount(() => {
    const cbJS = new ClipboardJS(".copy-button");
  });

  let sendBottomHovered = false;
  const onMouseEnter = () => {
    sendBottomHovered = true;
  };
  const onMouseLeave = () => {
    sendBottomHovered = false;
  };
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  class="relative mx-auto h-full max-h-screen w-full overflow-hidden"
  on:keydown={handleKeyDown}
>
  <div class="w-full">
    <div
      class="flex items-center justify-between border-b border-base-content/[.2] px-2 md:px-16"
    >
      <a
        class=""
        href="https://github.com/serge-chat/serge"
        target="_blank"
        rel="noopener noreferrer"
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 16 16"
          width="24"
          height="24"
        >
          <path
            class="fill-base-content"
            d="M8 0c4.42 0 8 3.58 8 8a8.013 8.013 0 0 1-5.45 7.59c-.4.08-.55-.17-.55-.38 0-.27.01-1.13.01-2.2 0-.75-.25-1.23-.54-1.48 1.78-.2 3.65-.88 3.65-3.95 0-.88-.31-1.59-.82-2.15.08-.2.36-1.02-.08-2.12 0 0-.67-.22-2.2.82-.64-.18-1.32-.27-2-.27-.68 0-1.36.09-2 .27-1.53-1.03-2.2-.82-2.2-.82-.44 1.1-.16 1.92-.08 2.12-.51.56-.82 1.28-.82 2.15 0 3.06 1.86 3.75 3.64 3.95-.23.2-.44.55-.51 1.07-.46.21-1.61.55-2.33-.66-.15-.24-.6-.83-1.23-.82-.67.01-.27.38.01.53.34.19.73.9.82 1.13.16.45.68 1.31 2.69.94 0 .67.01 1.3.01 1.49 0 .21-.15.45-.55.38A7.995 7.995 0 0 1 0 8c0-4.42 3.58-8 8-8Z"
          />
        </svg>
      </a>
      <div class="flex flex-col items-center justify-center">
        <h1 class="inline-block text-center text-base font-bold">
          Serge: {data.chat.params.model_path}
        </h1>
        <h4 class="text-center text-xs font-semibold">
          {startDate.toLocaleString("en-US")}
        </h4>
      </div>
      <div
        class="tooltip tooltip-left"
        data-tip="This will create a new chat session with the same parameters."
      >
        <button
          type="button"
          disabled={isLoading}
          class="btn-sm btn inline-block"
          class:loading={isLoading}
          on:click|preventDefault={() => createSameSession()}
        >
          New
        </button>
      </div>
    </div>
  </div>
  <div class="mb-11 h-[calc(97vh-12rem)] overflow-y-auto">
    <div class="h-max pb-32">
      {#each history as question, i}
        {#if question.type === "human"}
          <div
            class="chat chat-start border-t border-base-content/[.2] bg-base-300 px-10 py-4 md:px-16"
          >
            <div class="chat-image self-start pl-1 pt-1">
              <div
                class="mask mask-squircle online flex aspect-square w-[2.6rem] items-center justify-center overflow-hidden bg-gradient-to-b from-primary to-primary-focus"
              >
                <span>I</span>
              </div>
            </div>
            <div
              class="chat-bubble whitespace-normal break-words bg-base-300 text-base font-light text-base-content"
            >
              <!-- {question.data.content} -->
              <div class="w-full overflow-x-auto overflow-y-hidden break-words">
                {@html renderMarkdown(question.data.content)}
              </div>
            </div>
            {#if i === history.length - 1 && !isLoading}
              <div style="width: 100%; text-align: right;">
                <button
                  disabled={isLoading}
                  class="btn-ghost btn-sm btn"
                  on:click|preventDefault={() => deletePrompt(data.chat.id, i)}
                >
                  🗑️
                </button>
              </div>
            {/if}
          </div>
        {:else if question.type === "ai"}
          <div
            class="chat chat-start border-t border-base-content/[.2] bg-base-100 px-10 py-4 md:px-16"
          >
            <div class="chat-image self-start pl-1 pt-1">
              <div
                class="mask mask-squircle online flex aspect-square w-[2.6rem] items-center justify-center overflow-hidden bg-gradient-to-b from-primary to-primary-focus"
              >
                <span>AI</span>
              </div>
            </div>
            <div
              class="chat-bubble whitespace-normal break-words bg-base-100 text-base font-light text-base-content"
            >
              {#if question.data.content === ""}
                <div class="inline-block rounded bg-base-200 px-4 py-1">
                  <div class="dots-load aspect-square w-2 rounded-full" />
                </div>
              {/if}
              <!-- {question.data.content} -->
              <div class="w-full overflow-x-auto overflow-y-hidden break-words">
                {@html renderMarkdown(question.data.content)}
              </div>
            </div>
            {#if i === history.length - 1 && !isLoading}
              <div style="width: 100%; text-align: right;">
                <button
                  disabled={isLoading}
                  class="btn-ghost btn-sm btn"
                  on:click|preventDefault={() => deletePrompt(data.chat.id, i)}
                >
                  🗑️
                </button>
              </div>
            {/if}
          </div>
        {:else if question.type === "system"}
          <div
            class="text-md w-full px-10 py-8 text-center font-light md:px-16"
          >
            {question.data.content}
          </div>
        {/if}
      {/each}
    </div>
  </div>
  <div
    class="flex h-0 w-full flex-row items-center justify-center bg-base-100 px-0"
  >
    <div
      class="input-bordered input flex h-auto w-full max-w-3xl flex-row items-center justify-between rounded-lg bg-base-200 px-0 drop-shadow-md"
    >
      <textarea
        name="question"
        class="textarea h-10 flex-1 resize-y bg-[transparent] text-lg placeholder-base-content outline-0 ring-0 focus:outline-0"
        disabled={isLoading}
        placeholder="Message Serge..."
        bind:value={prompt}
      />
      <button
        on:mouseenter={onMouseEnter}
        on:mouseleave={onMouseLeave}
        type="submit"
        disabled={isLoading}
        class="btn-[transparent] btn h-10 w-14 rounded-l-none rounded-r-lg border-0 bg-[transparent] text-lg hover:bg-gradient-to-b hover:from-primary hover:to-primary-focus"
        class:loading={isLoading}
        on:click|preventDefault={askQuestion}
      >
        <svg
          xmlns="http://www.w3.org/2000/svg"
          viewBox="0 0 16 16"
          width="16"
          height="16"
        >
          <path
            class:fill-primary-content={sendBottomHovered}
            class:fill-base-content={!sendBottomHovered}
            d="M.989 8 .064 2.68a1.342 1.342 0 0 1 1.85-1.462l13.402 5.744a1.13 1.13 0 0 1 0 2.076L1.913 14.782a1.343 1.343 0 0 1-1.85-1.463L.99 8Zm.603-5.288L2.38 7.25h4.87a.75.75 0 0 1 0 1.5H2.38l-.788 4.538L13.929 8Z"
          />
        </svg>
      </button>
    </div>
  </div>
</div>
