<template>
  <div>
    <input ref="excel-upload-input" class="excel-upload-input" type="file" accept=".xlsx, .xls" @change="handleClick">
    <div class="drop" @drop="handleDrop" @dragover="handleDragover" @dragenter="handleDragover">
      Drop excel file here or
      <el-button :loading="loading" style="margin-left:16px;" size="mini" type="primary" @click="handleUpload">Browse</el-button>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop } from 'vue-property-decorator';
import XLSX from 'xlsx';

@Component
export default class UploadExcel extends Vue {
  @Prop()
  beforeUpload!: Function; // eslint-disable-line
  @Prop()
  onSuccess!: Function; // eslint-disable-line

  private loading: boolean = false;
  private excelData: { header: string[]|null, results: any[]|null } = {
    header: null,
    results: null,
  };

  generateData(data: { header: string[], results: any[] }) {
    this.excelData.header = data.header;
    this.excelData.results = data.results;
    this.onSuccess && this.onSuccess(this.excelData);
  }

  handleDrop(e: DragEvent) {
    e.stopPropagation();
    e.preventDefault();
    if (this.loading) return;
    if (e.dataTransfer == null) return;

    const files = e.dataTransfer.files;
    if (files.length !== 1) {
      this.$message.error('Only support uploading one file!');
      return;
    }
    const rawFile = files[0]; // only use files[0]

    if (!this.isExcel(rawFile)) {
      this.$message.error('Only supports upload .xlsx, .xls, .csv suffix files');
      return false;
    }
    this.upload(rawFile);
    e.stopPropagation();
    e.preventDefault();
  }

  handleDragover(e: DragEvent) {
    e.stopPropagation();
    e.preventDefault();
    if (e.dataTransfer != null) {
      e.dataTransfer.dropEffect = 'copy';
    }
  }

  handleUpload() {
    (this.$refs['excel-upload-input'] as any).click();
  }

  handleClick(e: Event) {
    const files = (e.target as any).files;
    const rawFile = files[0]; // only use files[0]
    if (!rawFile) return;
    this.upload(rawFile);
  }

  upload(rawFile: Blob) {
    (this.$refs['excel-upload-input'] as any).value = null; // fix can't select the same excel

    if (!this.beforeUpload) {
      this.readerData(rawFile);
      return;
    }
    const before = this.beforeUpload(rawFile);
    if (before) {
      this.readerData(rawFile);
    }
  }
  readerData(rawFile: Blob) {
    this.loading = true;
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = (e: Event) => {
        const data = (e.target as any).result;
        const workbook = XLSX.read(data, { type: 'array' });
        const firstSheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[firstSheetName];
        const header = this.getHeaderRow(worksheet);
        const results = XLSX.utils.sheet_to_json(worksheet);
        this.generateData({ header, results });
        this.loading = false;
        resolve();
      };
      reader.readAsArrayBuffer(rawFile);
    });
  }
  getHeaderRow(sheet: XLSX.Sheet): string[] {
    const headers = [];
    const range = XLSX.utils.decode_range(sheet['!ref'] as string);
    let C;
    const R = range.s.r;
    /* start in the first row */
    for (C = range.s.c; C <= range.e.c; ++C) { /* walk every column in the range */
      const cell = sheet[XLSX.utils.encode_cell({ c: C, r: R })];
      /* find the cell in the first row */
      let hdr = 'UNKNOWN ' + C; // <-- replace with your desired default
      if (cell && cell.t) hdr = XLSX.utils.format_cell(cell);
      headers.push(hdr);
    }
    return headers;
  }

  isExcel(file: { name: string }): boolean {
    return /\.(xlsx|xls|csv)$/.test(file.name);
  }
}
</script>

<style scoped>
.excel-upload-input{
  display: none;
  z-index: -9999;
}
.drop{
  border: 2px dashed #bbb;
  width: 600px;
  height: 160px;
  line-height: 160px;
  margin: 0 auto;
  font-size: 24px;
  border-radius: 5px;
  text-align: center;
  color: #bbb;
  position: relative;
}
</style>
