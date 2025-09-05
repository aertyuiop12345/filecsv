<template>
  <div class="min-h-screen bg-gray-100 py-10 px-4">
    <div class="max-w-6xl mx-auto bg-white rounded-lg shadow-lg p-6 space-y-8">
      <!-- Upload -->
      <div
        v-if="!fileData"
        class="flex flex-col items-center justify-center py-20"
      >
        <FileUpload
          name="file"
          accept=".csv"
          customUpload
          :auto="true"
          :chooseLabel="'Choose CSV'"
          @select="(e) => handleFileSelect(e.files[0])"
        />
        <p class="text-gray-500 mt-4">Please upload a CSV file to proceed.</p>
      </div>

      <!-- File Details and Table -->
      <div v-else class="space-y-8">
        <!-- Actions and File Info -->
        <div
          class="flex flex-col md:flex-row md:items-start md:justify-between gap-6"
        >
          <!-- Buttons -->
          <div class="flex gap-2">
            <Button
              label="Upload Another File"
              icon="pi pi-refresh"
              class="p-button-secondary"
              @click="handleReset"
            />
            <InputText
              placeholder="Search..."
              v-model="searchTerm"
              class="border border-gray-300 rounded px-3 py-2 w-full md:w-64"
            />
          </div>

          <!-- File Details Card -->
          <div
            class="bg-gray-50 border border-gray-200 rounded-lg p-4 shadow-sm text-sm text-gray-700 w-full md:w-auto"
          >
            <p class="mb-1">
              <span class="font-semibold">📄 File Name:</span>
              {{ fileData.file.name }}
            </p>
            <p class="mb-1">
              <span class="font-semibold">📦 Size:</span>
              {{ (fileData.file.size / 1024).toFixed(2) }} KB
            </p>
            <p class="mb-1">
              <span class="font-semibold">🔢 Rows:</span>
              {{ filteredRows.length }}
            </p>
            <p class="mb-1">
              <span class="font-semibold">📊 Columns:</span>
              {{ headers.length }}
            </p>
            <p>
              <span class="font-semibold">🕒 Date:</span>
              {{ new Date().toLocaleDateString() }}
            </p>
          </div>
        </div>
        <!-- Data Table -->
        <div class="overflow-x-auto rounded border border-gray-200 shadow-sm">
          <DataTable :value="paginatedRows" class="min-w-full">
            <Column
              v-for="(header, i) in headers"
              :key="i"
              :field="i.toString()"
              :header="header"
              :body="(row, column) => row[column.field]"
            />
          </DataTable>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";
import Button from "primevue/button";
import Card from "primevue/card";
import InputText from "primevue/inputtext";
import DataTable from "primevue/datatable";
import Column from "primevue/column";

interface FileData {
  file: File;
  content: string;
  parsedData: string[][];
}

const fileData = ref<FileData | null>(null);
const searchTerm = ref("");
const currentPage = ref(1);
const rowsPerPage = 50;

const toast = useToast();

// --- CSV Parsing ---
const parseCSV = (content: string): string[][] => {
  const lines = content.split("\n").filter((line) => line.trim());
  const result: string[][] = [];

  for (const line of lines) {
    const row: string[] = [];
    let current = "";
    let inQuotes = false;

    for (let i = 0; i < line.length; i++) {
      const char = line[i];
      if (char === '"') {
        if (inQuotes && line[i + 1] === '"') {
          current += '"';
          i++;
        } else {
          inQuotes = !inQuotes;
        }
      } else if (char === "," && !inQuotes) {
        row.push(current.trim());
        current = "";
      } else {
        current += char;
      }
    }
    row.push(current.trim());
    result.push(row);
  }

  return result;
};

// --- File Upload Handling ---
const handleFileSelect = async (file: File) => {
  if (!file.name.endsWith(".csv")) {
    toast.add({
      severity: "error",
      summary: "Error",
      detail: "Please select a CSV file",
    });
    return;
  }
  if (file.size > 10 * 1024 * 1024) {
    toast.add({
      severity: "error",
      summary: "Error",
      detail: "File size must be less than 10MB",
    });
    return;
  }

  const reader = new FileReader();
  reader.onload = (e) => {
    const content = e.target?.result as string;
    const parsedData = parseCSV(content);
    fileData.value = { file, content, parsedData };
    currentPage.value = 1;
  };
  reader.readAsText(file);
};

// --- Filter & Pagination ---
const headers = computed(() => fileData.value?.parsedData[0] || []);
const rows = computed(() => fileData.value?.parsedData.slice(1) || []);

const filteredRows = computed(() =>
  rows.value.filter((row) =>
    row.some((cell) =>
      cell.toLowerCase().includes(searchTerm.value.toLowerCase())
    )
  )
);

const paginatedRows = computed(() => {
  const start = (currentPage.value - 1) * rowsPerPage;
  return filteredRows.value.slice(start, start + rowsPerPage);
});

const totalPages = computed(() =>
  Math.ceil(filteredRows.value.length / rowsPerPage)
);

const handleReset = () => {
  fileData.value = null;
  searchTerm.value = "";
  currentPage.value = 1;
};
</script>
