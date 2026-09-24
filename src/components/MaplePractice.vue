<template>
  <div class="reward-sorter">
    <h2>序號整理工具</h2>
    <div class="layout">
      <!-- 輸入區 -->
      <div class="input-area">
        <label for="rawInput">貼上原始資料：</label>
        <textarea
          id="rawInput"
          v-model="rawInput"
          placeholder="貼上資料（範例：白金神奇剪刀	MYNY8WKK...	2026/9/16 22:32）"
          rows="15"
        ></textarea>
        <button @click="processData">整理資料</button>
      </div>

      <!-- 輸出區 -->
      <div class="output-area">
        <label for="processedOutput">整理後資料：</label>
        <textarea
          id="processedOutput"
          v-model="processedOutput"
          readonly
          rows="15"
          placeholder="整理後的資料會顯示在這裡..."
        ></textarea>
        <button @click="copyOutput" :disabled="!processedOutput">複製結果</button>
      </div>
    </div>

    <!-- 預覽表格 -->
    <div class="table-preview" v-if="parsedList.length > 0">
      <h3>資料預覽</h3>
      <table>
        <thead>
          <tr>
            <th>道具名稱</th>
            <th>序號</th>
            <th>獲得時間</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in parsedList" :key="index">
            <td>{{ item.name }}</td>
            <td>{{ item.serial }}</td>
            <td>{{ item.formattedDateTime }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';

// 定義資料的介面
interface RewardItem {
  name: string;
  serial: string;
  formattedDateTime: string;
}

const rawInput = ref('');
const processedOutput = ref('');
const parsedList = ref<RewardItem[]>([]);

const processData = () => {
  // 將字串依換行符號切割，移除前後空白並過濾掉空行
  const lines = rawInput.value.split('\n').map(line => line.trim()).filter(line => line !== '');
  const results: RewardItem[] = [];
  
  for (const line of lines) {
    // 略過表頭列
    if (line.includes('道具名稱') || line.includes('序號')) {
      continue;
    }

    // 優先以 Tab 切割資料，若使用者複製時 Tab 變成空白，則退而求其次用兩個以上的空白切割
    let parts = line.split('\t');
    if (parts.length < 3) {
      parts = line.split(/\s{2,}/);
    }

    // 若確認有成功切分出至少 3 個欄位，就推入陣列
    if (parts.length >= 3) {
      results.push({
        name: parts[0].trim(),
        serial: parts[1].trim(),
        // 為了防止日期與時間中間的單一空白被意外切斷，將後面的部分全部重新組合
        formattedDateTime: parts.slice(2).join(' ').trim()
      });
    }
  }

  // 排序規則：
  // 1. 道具名稱自然排序 (localeCompare 搭配 numeric: true)
  // 2. 獲得時間順序 (將時間字串轉為數值比較，由舊至新)
  results.sort((a, b) => {
    const nameCompare = a.name.localeCompare(b.name, undefined, { numeric: true });
    if (nameCompare !== 0) {
      return nameCompare;
    }

    // 當名稱相同時，進行時間比較
    const timeA = new Date(a.formattedDateTime).getTime();
    const timeB = new Date(b.formattedDateTime).getTime();
    
    // 防呆處理：若時間格式無效（例如 NaN），則回傳 0 維持原有順序
    if (isNaN(timeA) || isNaN(timeB)) return 0;
    
    return timeA - timeB; // 升冪排序（數值小/舊的在前，數值大/新的在後）
  });

  parsedList.value = results; 
  generateOutputText(results);
};

const generateOutputText = (data: RewardItem[]) => {
  if (data.length === 0) {
    processedOutput.value = '';
    return; // 沒有資料時直接中斷並回傳空值
  }

  const headers = ['道具名稱', '序號', '獲得時間'];
  const rows = data.map(item => 
    `${item.name}\t${item.serial}\t${item.formattedDateTime}`
  );

  processedOutput.value = [headers.join('\t'), ...rows].join('\n');
};

const copyOutput = async () => {
  try {
    await navigator.clipboard.writeText(processedOutput.value);
    alert('已成功複製到剪貼簿！');
  } catch (err) {
    alert('複製失敗，請手動框選文字複製。');
  }
};
</script>

<style scoped>
.reward-sorter {
  font-family: sans-serif;
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
}

.layout {
  display: flex;
  gap: 20px;
  margin-bottom: 20px;
}

.input-area, .output-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

textarea {
  width: 100%;
  resize: vertical;
  padding: 12px;
  font-family: monospace;
  font-size: 14px;
  border: 1px solid #ccc;
  border-radius: 4px;
  white-space: pre;
}

button {
  padding: 10px 16px;
  cursor: pointer;
  background-color: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 16px;
  transition: background-color 0.2s;
}

button:hover:not(:disabled) {
  background-color: #45a049;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

.table-preview {
  margin-top: 30px;
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

th, td {
  border: 1px solid #e0e0e0;
  padding: 10px;
  text-align: left;
}

th {
  background-color: #f5f5f5;
  font-weight: bold;
}
</style>