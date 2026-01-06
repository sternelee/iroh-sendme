<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue";
import { listen } from "@tauri-apps/api/event";
import { open } from "@tauri-apps/plugin-dialog";
import { send_file, receive_file, cancel_transfer, get_transfers } from "@/lib/commands";
import Button from "@/components/ui/button/Button.vue";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import {
  Popover,
  PopoverContent,
  PopoverTrigger,
} from "@/components/ui/popover";
import { Progress } from "@/components/ui/progress";
import { Loader2, FolderOpen } from "lucide-vue-next";

// Types
interface Transfer {
  id: string;
  transfer_type: string;
  path: string;
  status: string;
  created_at: number;
}

interface ProgressData {
  transfer_id: string;
  [key: string]: any;
}

interface ProgressUpdate {
  event_type: string;
  data: ProgressData & { transfer_id: string };
}

// State
const activeTab = ref("send");
const transfers = ref<Transfer[]>([]);

// Send state
const sendPath = ref("");
const sendTicketType = ref("relay_and_addresses");
const sendTicket = ref("");
const isSending = ref(false);

// Receive state
const receiveTicket = ref("");
const receiveOutputDir = ref("");
const isReceiving = ref(false);

// Progress state
const progressData = ref<Record<string, ProgressData>>({});
const unlisten = ref<(() => void) | null>(null);

// Ticket types
const ticketTypes = [
  { value: "id", label: "ID Only", description: "Smallest ticket, requires DNS" },
  { value: "relay", label: "Relay", description: "Uses relay server" },
  { value: "addresses", label: "Addresses", description: "Direct addresses" },
  {
    value: "relay_and_addresses",
    label: "Relay + Addresses",
    description: "Both relay and direct addresses",
  },
];

onMounted(async () => {
  await loadTransfers();
  // Listen for progress events
  unlisten.value = await listen<ProgressUpdate>("progress", (event) => {
    const { transfer_id, ...data } = event.payload.data;
    progressData.value[transfer_id] = { transfer_id, ...data };
  });
});

onUnmounted(() => {
  if (unlisten.value) {
    unlisten.value();
  }
});

async function loadTransfers() {
  try {
    transfers.value = await get_transfers();
  } catch (e) {
    console.error("Failed to load transfers:", e);
  }
}

async function handleSend() {
  if (!sendPath.value) {
    return;
  }

  isSending.value = true;
  sendTicket.value = "";

  try {
    const result = await send_file({
      path: sendPath.value,
      ticket_type: sendTicketType.value,
    });
    sendTicket.value = result;
    await loadTransfers();
  } catch (e) {
    console.error("Send failed:", e);
    alert(`Send failed: ${e}`);
  } finally {
    isSending.value = false;
  }
}

async function handleReceive() {
  if (!receiveTicket.value) {
    return;
  }

  isReceiving.value = true;

  try {
    await receive_file({
      ticket: receiveTicket.value,
      output_dir: receiveOutputDir.value || undefined,
    });
    await loadTransfers();
    receiveTicket.value = "";
  } catch (e) {
    console.error("Receive failed:", e);
    alert(`Receive failed: ${e}`);
  } finally {
    isReceiving.value = false;
  }
}

async function handleCancel(id: string) {
  try {
    await cancel_transfer(id);
    await loadTransfers();
  } catch (e) {
    console.error("Cancel failed:", e);
  }
}

function getTransferStatus(status: string): { label: string; color: string } {
  const s = status.toLowerCase();
  if (s.includes("error")) return { label: "Error", color: "text-red-500" };
  if (s.includes("cancel")) return { label: "Cancelled", color: "text-yellow-500" };
  if (s.includes("complete")) return { label: "Completed", color: "text-green-500" };
  if (s.includes("serving")) return { label: "Serving", color: "text-blue-500" };
  return { label: status, color: "text-gray-500" };
}

function formatFileSize(bytes: number): string {
  if (bytes === 0) return "0 B";
  const k = 1024;
  const sizes = ["B", "KB", "MB", "GB", "TB"];
  const i = Math.floor(Math.log(bytes) / Math.log(k));
  return `${(bytes / Math.pow(k, i)).toFixed(1)} ${sizes[i]}`;
}

function formatDate(timestamp: number): string {
  return new Date(timestamp * 1000).toLocaleString();
}

async function copyToClipboard(text: string) {
  await navigator.clipboard.writeText(text);
}

// File picker functions
async function selectFile() {
  try {
    const selected = await open({
      multiple: false,
      directory: false,
    });
    if (selected && typeof selected === "string") {
      sendPath.value = selected;
    }
  } catch (e) {
    console.error("Failed to select file:", e);
  }
}

async function selectDirectory() {
  try {
    const selected = await open({
      multiple: false,
      directory: true,
    });
    if (selected && typeof selected === "string") {
      sendPath.value = selected;
    }
  } catch (e) {
    console.error("Failed to select directory:", e);
  }
}

async function selectOutputDirectory() {
  try {
    const selected = await open({
      multiple: false,
      directory: true,
    });
    if (selected && typeof selected === "string") {
      receiveOutputDir.value = selected;
    }
  } catch (e) {
    console.error("Failed to select output directory:", e);
  }
}

function getDisplayName(path: string): string {
  if (!path) return "";
  const parts = path.split(/[/\\]/);
  return parts[parts.length - 1] || path;
}
</script>

<template>
  <main class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 dark:from-slate-950 dark:to-slate-900">
    <div class="container mx-auto px-4 py-8 max-w-4xl">
      <!-- Header -->
      <header class="text-center mb-8">
        <h1 class="text-4xl font-bold text-slate-900 dark:text-slate-50 mb-2">
          Sendme
        </h1>
        <p class="text-slate-600 dark:text-slate-400">
          Peer-to-peer file transfer powered by iroh
        </p>
      </header>

      <!-- Main Tabs -->
      <Tabs v-model="activeTab" class="space-y-6">
        <TabsList class="grid w-full grid-cols-2 max-w-md mx-auto">
          <TabsTrigger value="send">Send</TabsTrigger>
          <TabsTrigger value="receive">Receive</TabsTrigger>
        </TabsList>

        <!-- Send Tab -->
        <TabsContent value="send" class="space-y-4">
          <div class="bg-white dark:bg-slate-800 rounded-lg shadow-lg p-6 space-y-4">
            <h2 class="text-xl font-semibold text-slate-900 dark:text-slate-50">
              Send Files
            </h2>

            <!-- File Path Input -->
            <div class="space-y-2">
              <Label for="send-path">File or Directory</Label>
              <div class="flex gap-2">
                <Input
                  id="send-path"
                  v-model="sendPath"
                  placeholder="Select a file or directory..."
                  :disabled="isSending"
                  class="flex-1"
                  readonly
                />
                <Button
                  type="button"
                  @click="selectFile"
                  :disabled="isSending"
                  variant="outline"
                  size="icon"
                  title="Select File"
                >
                  <FolderOpen class="h-4 w-4" />
                </Button>
                <Button
                  type="button"
                  @click="selectDirectory"
                  :disabled="isSending"
                  variant="outline"
                  size="icon"
                  title="Select Directory"
                >
                  <FolderOpen class="h-4 w-4" />
                </Button>
              </div>
              <p v-if="sendPath" class="text-xs text-slate-500 dark:text-slate-400">
                Selected: <span class="font-mono">{{ getDisplayName(sendPath) }}</span>
              </p>
            </div>

            <!-- Ticket Type Selector -->
            <div class="space-y-2">
              <Label>Ticket Type</Label>
              <Popover>
                <PopoverTrigger as-child>
                  <Button
                    variant="outline"
                    class="w-full justify-between"
                    :disabled="isSending"
                  >
                    {{
                      ticketTypes.find((t) => t.value === sendTicketType)?.label ||
                      "Select ticket type"
                    }}
                    <span class="text-xs text-slate-500 ml-2">▼</span>
                  </Button>
                </PopoverTrigger>
                <PopoverContent class="w-56 p-0">
                  <div class="p-2">
                    <button
                      v-for="type in ticketTypes"
                      :key="type.value"
                      @click="sendTicketType = type.value"
                      class="w-full text-left px-3 py-2 rounded hover:bg-slate-100 dark:hover:bg-slate-700 transition-colors"
                      :class="{
                        'bg-slate-100 dark:bg-slate-700':
                          sendTicketType === type.value,
                      }"
                    >
                      <div class="font-medium text-sm">{{ type.label }}</div>
                      <div class="text-xs text-slate-500 dark:text-slate-400">
                        {{ type.description }}
                      </div>
                    </button>
                  </div>
                </PopoverContent>
              </Popover>
            </div>

            <!-- Send Button -->
            <Button
              @click="handleSend"
              :disabled="!sendPath || isSending"
              class="w-full"
              size="lg"
            >
              <Loader2 v-if="isSending" class="mr-2 h-4 w-4 animate-spin" />
              {{ isSending ? "Preparing..." : "Send File" }}
            </Button>

            <!-- Ticket Display -->
            <div v-if="sendTicket" class="mt-4 p-4 bg-slate-50 dark:bg-slate-900 rounded-lg">
              <Label class="text-sm text-slate-600 dark:text-slate-400 mb-2 block">
                Share this ticket to receive files:
              </Label>
              <div class="p-3 bg-white dark:bg-slate-800 rounded border border-slate-200 dark:border-slate-700 break-all text-sm font-mono">
                {{ sendTicket }}
              </div>
              <Button
                @click="copyToClipboard(sendTicket)"
                variant="outline"
                size="sm"
                class="mt-2"
              >
                Copy Ticket
              </Button>
            </div>
          </div>
        </TabsContent>

        <!-- Receive Tab -->
        <TabsContent value="receive" class="space-y-4">
          <div class="bg-white dark:bg-slate-800 rounded-lg shadow-lg p-6 space-y-4">
            <h2 class="text-xl font-semibold text-slate-900 dark:text-slate-50">
              Receive Files
            </h2>

            <!-- Ticket Input -->
            <div class="space-y-2">
              <Label for="receive-ticket">Ticket</Label>
              <Input
                id="receive-ticket"
                v-model="receiveTicket"
                placeholder="Paste the ticket here..."
                :disabled="isReceiving"
              />
            </div>

            <!-- Output Directory (Optional) -->
            <div class="space-y-2">
              <Label for="output-dir">Output Directory (Optional)</Label>
              <div class="flex gap-2">
                <Input
                  id="output-dir"
                  v-model="receiveOutputDir"
                  placeholder="Select output directory..."
                  :disabled="isReceiving"
                  class="flex-1"
                  readonly
                />
                <Button
                  type="button"
                  @click="selectOutputDirectory"
                  :disabled="isReceiving"
                  variant="outline"
                  size="icon"
                  title="Select Directory"
                >
                  <FolderOpen class="h-4 w-4" />
                </Button>
              </div>
              <p v-if="receiveOutputDir" class="text-xs text-slate-500 dark:text-slate-400">
                Selected: <span class="font-mono">{{ getDisplayName(receiveOutputDir) }}</span>
              </p>
            </div>

            <!-- Receive Button -->
            <Button
              @click="handleReceive"
              :disabled="!receiveTicket || isReceiving"
              class="w-full"
              size="lg"
            >
              <Loader2 v-if="isReceiving" class="mr-2 h-4 w-4 animate-spin" />
              {{ isReceiving ? "Receiving..." : "Receive File" }}
            </Button>
          </div>
        </TabsContent>
      </Tabs>

      <!-- Transfers List -->
      <div class="mt-8 bg-white dark:bg-slate-800 rounded-lg shadow-lg p-6">
        <div class="flex items-center justify-between mb-4">
          <h2 class="text-xl font-semibold text-slate-900 dark:text-slate-50">
            Transfers
          </h2>
          <Button @click="loadTransfers" variant="outline" size="sm">
            Refresh
          </Button>
        </div>

        <div v-if="transfers.length === 0" class="text-center py-8 text-slate-500">
          No transfers yet
        </div>

        <div v-else class="space-y-4">
          <div
            v-for="transfer in transfers"
            :key="transfer.id"
            class="p-4 border border-slate-200 dark:border-slate-700 rounded-lg space-y-3"
          >
            <div class="flex items-start justify-between">
              <div class="flex-1">
                <div class="flex items-center gap-2">
                  <span
                    class="text-xs font-medium px-2 py-1 rounded"
                    :class="{
                      'bg-blue-100 text-blue-700 dark:bg-blue-900 dark:text-blue-300':
                        transfer.transfer_type === 'send',
                      'bg-green-100 text-green-700 dark:bg-green-900 dark:text-green-300':
                        transfer.transfer_type === 'receive',
                    }"
                  >
                    {{ transfer.transfer_type.toUpperCase() }}
                  </span>
                  <span
                    class="text-sm font-medium"
                    :class="
                      getTransferStatus(transfer.status).color
                    "
                  >
                    {{ getTransferStatus(transfer.status).label }}
                  </span>
                </div>
                <div class="text-sm text-slate-600 dark:text-slate-400 mt-1 truncate max-w-md">
                  {{ transfer.path }}
                </div>
                <div class="text-xs text-slate-500 dark:text-slate-500 mt-1">
                  {{ formatDate(transfer.created_at) }}
                </div>
              </div>

              <Button
                v-if="!transfer.status.includes('complete') &&
                      !transfer.status.includes('error') &&
                      !transfer.status.includes('cancel')"
                @click="handleCancel(transfer.id)"
                variant="destructive"
                size="sm"
              >
                Cancel
              </Button>
            </div>

            <!-- Progress Bar -->
            <div v-if="progressData[transfer.id]" class="space-y-2">
              <Progress
                v-if="progressData[transfer.id].progress?.type === 'downloading'"
                :value="
                  (progressData[transfer.id].progress.offset /
                    progressData[transfer.id].progress.total) *
                  100
                "
                class="h-2"
              />
              <div class="text-xs text-slate-500 dark:text-slate-400">
                <template v-if="progressData[transfer.id].name">
                  {{ progressData[transfer.id].name }}
                </template>
                <template v-if="progressData[transfer.id].progress?.type === 'downloading'">
                  - {{
                    formatFileSize(progressData[transfer.id].progress.offset) +
                    " / " +
                    formatFileSize(progressData[transfer.id].progress.total)
                  }}
                </template>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
/* Custom scrollbar */
:deep(*) {
  scrollbar-width: thin;
  scrollbar-color: rgb(203 213 225) transparent;
}

:deep(*):-webkit-scrollbar {
  width: 8px;
  height: 8px;
}

:deep(*):-webkit-scrollbar-track {
  background: transparent;
}

:deep(*):-webkit-scrollbar-thumb {
  background-color: rgb(203 213 225);
  border-radius: 4px;
}
</style>
