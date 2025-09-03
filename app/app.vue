<template>
  <div>
    <div v-if="!fileData">
      <FileUpload
        name="file"
        accept=".csv"
        customUpload
        :auto="true"
        :chooseLabel="'Choose CSV'"
        @select="(e) => handleFileSelect(e.files[0])"
      />
    </div>

    <div v-else class="space-y-4">
      <div class="flex gap-2">
        <Button
          label="Upload Another File"
          icon="pi pi-refresh"
          @click="handleReset"
        />
        <InputText placeholder="Search..." v-model="searchTerm" />
      </div>

      <p>
        <strong>File:</strong> {{ fileData.file.name }} ({{
          (fileData.file.size / 1024).toFixed(2)
        }}
        KB)
      </p>
      <p>
        <strong>Rows:</strong> {{ filteredRows.length }} | <br />
        <strong>Columns:</strong> {{ headers.length }} | <br />
        <strong>Date:</strong>
        {{ new Date().toLocaleDateString() }}
      </p>

      <DataTable :value="paginatedRows">
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

<style scoped>
.space-y-4 > * + * {
  margin-top: 1rem;
}
</style>
