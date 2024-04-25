<script>
  import { socket } from "$lib/api";
  import { agentState, messages } from "$lib/store";
  import { calculateTokens } from "$lib/token";
  import { Icons } from "../icons";
  import { X } from 'lucide-svelte';

  let isAgentActive = false;
  // ---- BEGIN UPDATE ---- //
  let files = [];
  let previews = [];

  // function handleFileSelect(event) {
  //   files = Array.from(event.target.files);
  //   previews = files.map(file => ({
  //     url: URL.createObjectURL(file),
  //     name: file.name,
  //   }));
  // }

  let fileID = 0; // Counter for unique ID
  function handleFileSelect(event) {
    const newFiles = Array.from(event.target.files);
    const newPreviews = newFiles.map(file => {
      const fileType = file.type;
      let icon;
      if (fileType.startsWith('image')) {
        icon = URL.createObjectURL(file);  // Display image preview for image files
      } else if (fileType.includes('excel') || fileType.includes('spreadsheetml')) {
        icon = Icons.Sheet;  // Display custom Excel icon for Excel files
      } else {
        icon = Icons.File;  // Display custom document icon for all other file types
      }
      return {
        id: fileID++, // Assign unique ID
        url: icon,
        name: file.name,
        isImage: fileType.startsWith('image')
      };
    });

    files = [...files, ...newFiles];  // Concatenate new files to existing files
    previews = [...previews, ...newPreviews];  // Concatenate new previews to existing previews
  }

  function removeFile(index) {
    console.log('Attempting to remove index:', index, 'Current previews:', previews);
    if (previews[index]) {
        if (previews[index].isImage) {
            URL.revokeObjectURL(previews[index].url);
        }
        let newPreviews = previews.slice();
        newPreviews.splice(index, 1);
        previews = newPreviews; // Assign new array to previews
    }
    console.log('After removing index file, current previews:', previews);
  }

  function triggerFileInput() {
      document.getElementById('file-input').click();
  }

    // ---- END UPDATE ---- //

    if ($agentState !== null) {
      isAgentActive = $agentState.agent_is_active;
    }

    let messageInput = "";
    async function handleSendMessage() {
      const projectName = localStorage.getItem("selectedProject");
      const selectedModel = localStorage.getItem("selectedModel");
      const searchEngine = localStorage.getItem("selectedSearchEngine");

      if (!projectName) {
        alert("Please select a project first!");
        return;
      }
      if (!selectedModel) {
        alert("Please select a model first!");
        return;
      }

      // ---- BEGIN UPDATE ---- //
      if (messageInput.trim() !== "" && !isAgentActive) {
        const action = $messages.length === 0 ? 'execute_agent' : 'continue';
        socket.emit("user-message", {
          action: action,
          message: messageInput,
          base_model: selectedModel,
          project_name: projectName,
          search_engine: searchEngine,
          files: files.map(file => ({ name: file.name, type: file.type })) // Send basic file info if needed
        });
        messageInput = "";
        files = [];
        previews = [];
      }
      // ---- END UPDATE ---- //
    }

  function setTokenSize(event) {
    const prompt = event.target.value;
    let tokens = calculateTokens(prompt);
    document.querySelector(".token-count").textContent = `${tokens}`;
  }
</script>

<div class="expandable-input relative">
  <div class="py-3 px-1 rounded-md text-xs">
    Agent status:
    {#if $agentState !== null}
      {#if $agentState.agent_is_active}
        <span class="text-green-500">Active</span>
      {:else}
        <span class="text-orange-600">Inactive</span>
      {/if}
    {:else}
      Deactive
    {/if}
  </div>
  <!-- BEGIN UPDATE -->
  <div class="file-upload my-2">
    <button class="upload-button bg-secondary rounded p-2 w-[40px] h-[40px]" on:click={triggerFileInput}>
      {@html Icons.Upload} <!-- Custom Upload Icon -->
    </button>
    <input type="file" id="file-input" multiple on:change={handleFileSelect} class="hidden">
    <div class="previews">
    {#each previews as preview, index (preview.id)}
      <div class="preview relative bg-secondary rounded p-2">
        {#if preview.isImage}
          <img src={preview.url} alt={preview.name}>
        {:else}
          {@html preview.url} <!-- Display SVG icon -->
        {/if}
        <button on:click={() => removeFile(index)} class="absolute -top-2 -right-2 text-red-500"><X class="size-4"/></button>
      </div>
    {/each}
  </div>
  </div>
  <!-- END UPDATE -->
  <div class="w-full p-4 font-medium rounded-xl h-28 bg-secondary flex">
    <textarea
      id="message-input"
      class="flex-1 bg-transparent focus:text-foreground outline-none"
      placeholder="Type your message..."
      bind:value={messageInput}
      on:input={setTokenSize}
      on:keydown={(e) => {
        if (e.key === "Enter" && !e.shiftKey) {
          e.preventDefault();
          handleSendMessage();
        }
      }}
    ></textarea>
    <div class="flex flex-col">
      <p class="text-tertiary p-2 right-4 top-12">
        <span class="token-count">0</span>
      </p>
      <button 
        on:click={handleSendMessage}
        disabled={isAgentActive}
        class="text-secondary bg-primary p-2 right-4 bottom-6 rounded-full"
      >
      {@html Icons.CornerDownLeft} 
      </button>
    </div>
  </div>

</div>

<style>
  /* ---- BEGIN UPDATE ---- */
  .file-upload {
    display: flex;
    gap: 5px;
    align-items: center;
  }

  .hidden {
    display: none;
  }
  .upload-button {
    border: none;
    cursor: pointer;
  }

  .previews {
    display: flex;
    gap: 5px 10px;
    flex-wrap: wrap;
    margin-top: 10px;
    margin-bottom: 10px;
  }

  .preview {
    width: 50px; /* Adjust as necessary */
    height: auto;
  }

  /* ---- END UPDATE ---- */
  .expandable-input textarea {
    min-height: 60px;
    max-height: 200px;
    resize: none;
  }
</style>
