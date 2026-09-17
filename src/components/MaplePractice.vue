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
          placeholder="請在此貼上原始格式..."
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
            <th>獲得日</th>
            <th>時間</th>
            <th>獎勵等級</th>
            <th>獎勵名稱</th>
            <th>序號</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in parsedList" :key="index">
            <td>{{ item.date }}</td>
            <td>{{ item.time }}</td>
            <td>{{ item.level }}</td>
            <td>{{ item.name }}</td>
            <td>{{ item.serial }}</td>
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
  date: string;
  time: string;
  level: string;
  name: string;
  serial: string;
}

const rawInput = ref('');
const processedOutput = ref('');
const parsedList = ref<RewardItem[]>([]);

const processData = () => {
  // 將字串依換行符號切割，移除前後空白並過濾掉空行
  const lines = rawInput.value.split('\n').map(line => line.trim()).filter(line => line !== '');
  const results: RewardItem[] = [];
  
  let pendingDate = '';

  for (const line of lines) {
    // 略過表頭
    if (line.includes('獲得日') || line.includes('獎勵名稱') || line.includes('序號')) {
      continue;
    }

    const parts = line.split('\t');

    // 判斷是否為獨立的日期行 (例如 09/16)
    if (parts.length === 1 && /^\d{2}\/\d{2}$/.test(parts[0])) {
      pendingDate = parts[0];
    } else if (parts.length >= 4) {
      // 處理資料行：時間、等級、名稱、序號
      let date = pendingDate;
      let time = parts[0];

      // 預防極端情況：日期與時間在同一行且以空白分隔
      if (parts[0].includes(' ')) {
        const dt = parts[0].split(' ');
        date = dt[0];
        time = dt[1];
      }

      results.push({
        date,
        time,
        level: parts[1],
        name: parts[2],
        serial: parts[3]
      });
    }
  }

  // 排序規則：
  // 1. 獎勵名稱自然排序 (localeCompare 搭配 numeric: true 可正確判斷數字大小)
  // 2. 獲得日順序 (將日期與時間合併進行字串比較，由舊至新)
  results.sort((a, b) => {
    const nameCompare = a.name.localeCompare(b.name, undefined, { numeric: true });
    if (nameCompare !== 0) {
      return nameCompare; // 若名稱不同，直接回傳名稱的比較結果
    }

    // 名稱相同時，比較時間
    const timeA = `${a.date} ${a.time}`;
    const timeB = `${b.date} ${b.time}`;
    return timeA.localeCompare(timeB);
  });

  parsedList.value = results;
  generateOutputText(results);
};

const generateOutputText = (data: RewardItem[]) => {
  if (data.length === 0) {
    processedOutput.value = '';
    return; // 沒有資料時中斷並回傳
  }

  const headers = ['獲得日', '時間', '獎勵等級', '獎勵名稱', '序號'];
  const rows = data.map(item => 
    `${item.date}\t${item.time}\t${item.level}\t${item.name}\t${item.serial}`
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
  max-width: 1200px;
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