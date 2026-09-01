<template>
  <view :class="page === 'main' ? 'main-container' : 'container'">
    <!-- 显示当前登录的账号名 -->
    <view v-if="currentUsername" class="header-bar">
      <text>当前用户: {{ currentUsername }}</text>
    </view>
    <br>
    <!-- 新增：筛选功能菜单 -->
    <view v-if="page!== 'main'" class="filter-menu">
      <view class="input-group">
        <input v-model="filterKeyword" placeholder="输入筛选关键字" class="filter-input" />
        <!-- 修改为显示当前筛选的记录数 -->
        <br>
        <view class="filter-count">
          当前筛选记录数: {{ page === 'outbound'? filteredOutboundRecords.length : filteredInboundRecords.length }}
        </view>
      </view>
    </view>

    <!-- 主界面 -->
    <view v-if="page === 'main'" class="main-page">
      <view class="title">包装箱二维码扫码</view>
      <view class="buttons">
        <button @click="goToPage('outbound')" class="main-button">出库扫码 [Outbound]</button>
        <button @click="goToPage('inbound')" class="main-button">入库扫码 [Inbound]</button>
        <button @click="goToPage('verification')" class="main-button">记录核验 [Validation]</button>
      </view>

      <view class="fixed bottom-10 left-0 right-0 text-center bg-gray-200 p-2">
        <br><br>
      </view>
    </view>
    <!-- 出库记录页面 -->
    <view v-if="page === 'outbound'">
      <!-- 顶部操作栏 -->
      <view class="header-bar">
        <view class="title">出库</view>
        <view class="scan-btn" @click="startScan">
          <text class="scan-icon">[-]</text>
        </view>

        <view class="scan-count">已出库：{{ scanCount }} 箱</view>

        <view class="header-actions">
          <view 
            class="export-btn"
            :class="{ disabled: exportMode && !hasSelections }"
            @click="handleExport"
          >
            {{ exportMode? '完成导出' : '导出' }}
          </view>
          <view 
            v-if="exportMode"
            class="cancel-btn"
            @click="cancelExport"
          >
            取消
          </view>
        </view>
      </view>

      <!-- 记录列表 -->
      <scroll-view class="record-list" scroll-y>
        <view 
          v-for="(record, index) in filteredOutboundRecords" 
          :key="index"
          class="record-item"
          :class="{ highlight: record.highlight }"
        >
          <view 
            v-if="exportMode" 
            class="checkbox"
            @click="toggleSelect(index)"
          >
            <text v-if="record.selected" class="check-icon">✓</text>
          </view>
          
          <text class="content">【{{ record.time }}】【{{ record.location }}】</text>
          <view class="divider"></view>
          <text class="content">{{ record.content }}</text>

          <view v-if="!exportMode" class="action-btns">
            <text class="edit-btn" @click="confirmEdit('outbound', index)">✎</text>
            <text class="delete-btn" @click="confirmDelete('outbound', index)">×</text>
          </view>
        </view>

        <view v-if="!filteredOutboundRecords.length" class="empty-tip">
          暂无出库记录
        </view>
      </scroll-view>

      <!-- 删除确认弹窗 -->
      <view v-if="showDeleteConfirm" class="modal-mask">
        <view class="modal-content">
          <input
            v-model="deletePassword"
            class="password-input"
            placeholder="输入口令"
            placeholder-style="color: #999"
          />
          <view class="modal-btns">
            <text class="cancel-btn" @click="showDeleteConfirm = false">取消</text>
            <text class="confirm-btn" @click="handleDelete">确认</text>
          </view>
        </view>
      </view>

      <!-- 编辑确认弹窗 -->
      <view v-if="showEditConfirm" class="modal-mask">
        <view class="modal-content">
          <input
            v-model="editPassword"
            class="password-input"
            placeholder="输入口令"
            placeholder-style="color: #999"
          />
          <view class="modal-btns">
            <text class="cancel-btn" @click="showEditConfirm = false">取消</text>
            <text class="confirm-btn" @click="handleEdit">确认</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 入库记录页面 -->
    <view v-if="page === 'inbound'">
      <view class="header-bar">
        <view class="title">入库</view>
        <view class="scan-btn" @click="startScanInbound">
          <text class="scan-icon">[-]</text>
        </view>

        <view class="scan-count">已入库：{{ inboundCount }} 箱</view>
      </view>
      <scroll-view class="record-list" scroll-y>
        <view 
          v-for="(record, index) in filteredInboundRecords" 
          :key="index"
          class="record-item"
        >
          <text class="content">【{{ record.outboundTime }}出库】【{{ record.inboundTime }}入库】【{{ record.inboundLocation }}】</text>
          <view class="divider"></view>
          <text class="content">{{ record.content }}</text>
          <view v-if="!exportMode" class="action-btns">
            <text class="edit-btn" @click="confirmEdit('inbound', index)">✎</text>
            <text class="delete-btn" @click="confirmDelete('inbound', index)">×</text>
          </view>
        </view>
        <view v-if="!filteredInboundRecords.length" class="empty-tip">
          暂无入库记录
        </view>
      </scroll-view>
      <!-- 入库记录删除确认弹窗 -->
      <view v-if="showInboundDeleteConfirm" class="modal-mask">
        <view class="modal-content">
          <input
            v-model="inboundDeletePassword"
            class="password-input"
            placeholder="输入口令"
            placeholder-style="color: #999"
          />
          <view class="modal-btns">
            <text class="cancel-btn" @click="showInboundDeleteConfirm = false">取消</text>
            <text class="confirm-btn" @click="handleInboundDelete">确认</text>
          </view>
        </view>
      </view>
      <!-- 入库记录编辑确认弹窗 -->
      <view v-if="showInboundEditConfirm" class="modal-mask">
        <view class="modal-content">
          <input
            v-model="inboundEditPassword"
            class="password-input"
            placeholder="输入口令"
            placeholder-style="color: #999"
          />
          <view class="modal-btns">
            <text class="cancel-btn" @click="showInboundEditConfirm = false">取消</text>
            <text class="confirm-btn" @click="handleInboundEdit">确认</text>
          </view>
        </view>
      </view>
    </view>

    <!-- 核验页面 -->
    <view v-if="page === 'verification'">
      <view class="header-bar">
        <view class="title">记录核验</view>
        <span class="matched-count-text">通过箱数：{{ matchedCount }}</span>
      </view>
      <div class="flex justify-center w-full mx-auto">
        <button @click="verifyOutbound" class="verify-button bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded mr-2">出库核验</button>
        <button @click="verifyInbound" class="verify-button bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded mr-2">入库核验</button>
        <button @click="verifyRecords" class="verify-button bg-yellow-500 hover:bg-yellow-600 text-white font-bold py-2 px-4 rounded">匹配核验</button>
      </div>
      <view class="result" v-if="verificationResult">
        <div v-if="unmatchedOutbound.length > 0 || unmatchedInbound.length > 0">
          <div v-if="unmatchedOutbound.length > 0">
            <p>未入库的出库记录（共 {{ unmatchedOutbound.length }} 条）：</p>
            <p 
              v-for="(content, index) in unmatchedOutbound" 
              :key="index"
              :style="{ 
                backgroundColor: index % 2 === 1? '#f0f0f0' : 'white', 
                color: index % 2 === 0? 'darkred' : 'darkblue',
                fontSize: '16px',
                overflow: 'auto' 
              }"
            >
              {{ content }}
            </p>
          </div>
          <div v-if="unmatchedInbound.length > 0">
            <p>未出库的入库记录（共 {{ unmatchedInbound.length }} 条）：</p>
            <p 
              v-for="(content, index) in unmatchedInbound" 
              :key="index"
              :style="{ 
                backgroundColor: index % 2 === 1? '#f0f0f0' : 'white', 
                color: index % 2 === 0? 'darkred' : 'darkblue',
                fontSize: '16px',
                overflow: 'auto' 
              }"
            >
              {{ content }}
            </p>
          </div>
        </div>
        <div v-if="matchedCount > 0" class="matched-count">
          核验通过的箱数：{{ matchedCount }}
        </div>
      </view>
      <div v-if="outboundVerificationResult" class="result">
        <p>出库核验结果：</p>
        <p v-if="outboundMissingNumbers.length > 0">
          缺失的出库编号：{{ outboundMissingNumbers.join(', ') }}
        </p>
        <p v-else>出库记录编号连续，核验通过。</p>
      </div>
      <div v-if="inboundVerificationResult" class="result">
        <p>入库核验结果：</p>
        <p v-if="inboundMissingNumbers.length > 0">
          缺失的入库编号：{{ inboundMissingNumbers.join(', ') }}
        </p>
        <p v-else>入库记录编号连续，核验通过。</p>
      </div>
    </view>

    <!-- 返回按钮 -->
    <view v-if="page!== 'main'" class="back-btn" @click="goBack">返回</view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';

// 读取当前用户名
const currentUsername = ref(uni.getStorageSync('currentUsername'));

// 响应式状态
const page = ref('main');
const scanRecords = ref([]);
const inboundRecords = ref([]);
const scanCount = ref(0);
const inboundCount = ref(0); // 新增：入库箱数
const exportMode = ref(false);
const showDeleteConfirm = ref(false);
const showEditConfirm = ref(false);
const showInboundDeleteConfirm = ref(false);
const showInboundEditConfirm = ref(false);
const deletePassword = ref('');
const editPassword = ref('');
const inboundDeletePassword = ref('');
const inboundEditPassword = ref('');
const currentDeleteType = ref('');
const currentDeleteIndex = ref(-1);
const currentEditType = ref('');
const currentEditIndex = ref(-1);
const verificationResult = ref('');
const unmatchedOutbound = ref([]);
const unmatchedInbound = ref([]);
const matchedCount = ref(0);
// 新增：筛选相关状态
const filterKeyword = ref('');
const originalOutboundRecords = ref([]);
const originalInboundRecords = ref([]);
const isFiltering = ref(false);
// 新增：出库、入库核验相关状态
const outboundVerificationResult = ref('');
const inboundVerificationResult = ref('');
const outboundMissingNumbers = ref([]);
const inboundMissingNumbers = ref([]);

// 时间格式化方法
const formatTime = (date = new Date()) => {
  const pad = n => n.toString().padStart(2, '0');
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}`;
};

// 经纬度格式化方法
const formatLocation = (latitude, longitude) => {
  const formatCoord = (coord) => {
    const deg = Math.floor(coord);
    const min = Math.floor((coord - deg) * 60);
    const sec = ((coord - deg - min / 60) * 3600).toFixed(2);
    return `${deg}°${min}'${sec}"`;
  };
  return `${formatCoord(latitude)}:${formatCoord(longitude)}`;
};

// 计算属性
const hasSelections = computed(() => 
  scanRecords.value.some(r => r.selected)
);

// 新增：筛选后的出库记录
const filteredOutboundRecords = computed(() => {
  if (!filterKeyword.value) {
    return scanRecords.value;
  }
  return scanRecords.value.filter(record => 
    record.content.includes(filterKeyword.value)
  ).sort((a, b) => new Date(b.time) - new Date(a.time));
});

// 新增：筛选后的入库记录
const filteredInboundRecords = computed(() => {
  if (!filterKeyword.value) {
    return inboundRecords.value;
  }
  return inboundRecords.value.filter(record => 
    record.content.includes(filterKeyword.value)
  ).sort((a, b) => new Date(b.inboundTime) - new Date(a.inboundTime));
});

// 初始化加载数据
onMounted(() => {
  const savedOutboundData = uni.getStorageSync(`${currentUsername.value}_outboundData`);
  if (savedOutboundData) {
    scanRecords.value = savedOutboundData.records.map(r => ({
      ...r,
      time: r.time || formatTime(new Date(r.timestamp)),
      location: r.location || ''
    }));
    scanCount.value = savedOutboundData.count || 0;
    originalOutboundRecords.value = [...scanRecords.value];
  }
  const savedInboundData = uni.getStorageSync(`${currentUsername.value}_inboundData`);
  if (savedInboundData) {
    inboundRecords.value = savedInboundData;
    inboundCount.value = savedInboundData.length; // 新增：加载入库箱数
    originalInboundRecords.value = [...inboundRecords.value];
  }
});

// 数据存储
const saveOutboundData = () => {
  uni.setStorageSync(`${currentUsername.value}_outboundData`, {
    records: scanRecords.value,
    count: scanCount.value
  });
};

const saveInboundData = () => {
  uni.setStorageSync(`${currentUsername.value}_inboundData`, inboundRecords.value);
  inboundCount.value = inboundRecords.value.length; // 新增：更新入库箱数
};

// 切换页面
const goToPage = (newPage) => {
  page.value = newPage;
};

// 返回主界面
const goBack = () => {
  page.value = 'main';
};

// 获取经纬度
const getLocation = () => {
  return new Promise((resolve, reject) => {
    uni.getLocation({
      type: 'wgs84',
      success: (res) => {
        resolve(formatLocation(res.latitude, res.longitude));
      },
      fail: (err) => {
        reject(err);
      }
    });
  });
};

// 出库扫描功能
const startScan = async () => {
  try {
    const res = await new Promise((resolve, reject) => {
      uni.scanCode({
        onlyFromCamera: true,
        scanType: ['qrCode'],
        success: (res) => resolve(res),
        fail: (err) => reject(err)
      });
    });
    const location = await getLocation();
    handleScanResult(res.result, location);
  } catch (err) {
    if (err.errMsg.includes('scanCode:fail')) {
      uni.showToast({ title: '扫描失败', icon: 'none' });
    } else {
      uni.showToast({ title: '获取位置失败', icon: 'none' });
    }
  }
};

const handleScanResult = (result, location) => {
  if (!result.startsWith('uid')) {
    uni.showToast({ title: '无效二维码，无法记录', icon: 'none' });
    return;
  }
  const content = result.slice(3);
  const existing = scanRecords.value.find(r => r.content === content);
  
  if (existing) {
    existing.highlight = true;
    setTimeout(() => existing.highlight = false, 3000);
    uni.showToast({ title: '重复扫描，请检查！', icon: 'none' });
  } else {
    const newRecord = {
      content: content,
      time: formatTime(),
      timestamp: Date.now(),
      location: location,
      highlight: false,
      selected: false
    };
    scanRecords.value.unshift(newRecord);
    scanCount.value++;
    originalOutboundRecords.value = [...scanRecords.value];
    nextTick(() => {
      saveOutboundData();
      uni.showToast({ title: '出库扫码成功', icon: 'success' });
    });
  }
};

// 入库扫描功能
const startScanInbound = async () => {
  try {
    const res = await new Promise((resolve, reject) => {
      uni.scanCode({
        onlyFromCamera: true,
        scanType: ['qrCode'],
        success: (res) => resolve(res),
        fail: (err) => reject(err)
      });
    });
    const location = await getLocation();
    handleInboundScanResult(res.result, location);
  } catch (err) {
    if (err.errMsg.includes('scanCode:fail')) {
      uni.showToast({ title: '扫码失败', icon: 'none' });
    } else {
      uni.showToast({ title: '获取位置失败', icon: 'none' });
    }
  }
};

const handleInboundScanResult = (result, location) => {
  if (!result.startsWith('uid')) {
    uni.showToast({ title: '无效二维码', icon: 'none' });
    return;
  }
  const content = result.slice(3);
  const outboundRecord = scanRecords.value.find(r => r.content === content);
  const isDuplicate = inboundRecords.value.some(r => r.content === content);

  if (isDuplicate) {
    uni.showToast({ title: '该记录已扫码入库', icon: 'none' });
    return;
  }

  if (outboundRecord) {
    const newRecord = {
      content: content,
      outboundTime: outboundRecord.time,
      inboundTime: formatTime(),
      inboundLocation: location
    };
    inboundRecords.value.unshift(newRecord);
    originalInboundRecords.value = [...inboundRecords.value];
    nextTick(() => {
      saveInboundData();
      uni.showToast({ title: '入库扫描成功', icon: 'success' });
    });
  } else {
    uni.showToast({ title: '未找到对应的出库记录', icon: 'none' });
  }
};

// 记录操作
const confirmEdit = (type, index) => {
  if (type === 'inbound') {
    currentEditType.value = type;
    currentEditIndex.value = index;
    showInboundEditConfirm.value = true;
    inboundEditPassword.value = '';
  } else {
    currentEditType.value = type;
    currentEditIndex.value = index;
    showEditConfirm.value = true;
    editPassword.value = '';
  }
};

const handleEdit = () => {
  if (editPassword.value === '编辑') {
    if (currentEditType.value === 'outbound') {
      uni.showModal({
        title: '编辑记录',
        content: scanRecords.value[currentEditIndex.value].content,
        editable: true,
        success: (res) => {
          if (res.confirm && res.content) {
            scanRecords.value[currentEditIndex.value].content = res.content.trim();
            originalOutboundRecords.value = [...scanRecords.value];
            saveOutboundData();
          }
        }
      });
    }
    uni.showToast({ title: '编辑成功' });
  } else {
    uni.showToast({ title: '口令错误', icon: 'none' });
  }
  showEditConfirm.value = false;
};

const handleInboundEdit = () => {
  if (inboundEditPassword.value === '编辑') {
    uni.showModal({
      title: '编辑记录',
      content: inboundRecords.value[currentEditIndex.value].content,
      editable: true,
      success: (res) => {
        if (res.confirm && res.content) {
          inboundRecords.value[currentEditIndex.value].content = res.content.trim();
          originalInboundRecords.value = [...inboundRecords.value];
          saveInboundData();
        }
      }
    });
    uni.showToast({ title: '编辑成功' });
  } else {
    uni.showToast({ title: '口令错误', icon: 'none' });
  }
  showInboundEditConfirm.value = false;
};

const confirmDelete = (type, index) => {
  if (type === 'inbound') {
    currentDeleteType.value = type;
    currentDeleteIndex.value = index;
    showInboundDeleteConfirm.value = true;
    inboundDeletePassword.value = '';
  } else {
    currentDeleteType.value = type;
    currentDeleteIndex.value = index;
    showDeleteConfirm.value = true;
    deletePassword.value = '';
  }
};

const handleDelete = () => {
  if (deletePassword.value === '删除') {
    if (currentDeleteType.value === 'outbound') {
      scanRecords.value.splice(currentDeleteIndex.value, 1);
      scanCount.value = Math.max(0, scanCount.value - 1);
      originalOutboundRecords.value = [...scanRecords.value];
      saveOutboundData();
    }
    uni.showToast({ title: '删除成功' });
  } else {
    uni.showToast({ title: '口令错误', icon: 'none' });
  }
  showDeleteConfirm.value = false;
};

const handleInboundDelete = () => {
  if (inboundDeletePassword.value === '删除') {
    inboundRecords.value.splice(currentDeleteIndex.value, 1);
    originalInboundRecords.value = [...inboundRecords.value];
    saveInboundData();
    uni.showToast({ title: '删除成功' });
  } else {
    uni.showToast({ title: '口令错误', icon: 'none' });
  }
  showInboundDeleteConfirm.value = false;
};

// 导出功能
const toggleSelect = (index) => {
  scanRecords.value[index].selected = !scanRecords.value[index].selected;
};

const handleExport = () => {
  if (!exportMode.value) {
    exportMode.value = true;
    return;
  }

  if (!hasSelections.value) {
    uni.showToast({ title: '请选择要导出的记录', icon: 'none' });
    return;
  }

  exportToTxt();
};

const cancelExport = () => {
  exportMode.value = false;
  scanRecords.value.forEach(r => r.selected = false);
};

const exportToTxt = () => {
  const selectedRecords = scanRecords.value
    .filter(r => r.selected)
    .map(r => `【${r.time}】【${r.location}】\n---\n${r.content}`);

  if (selectedRecords.length === 0) return;

  const content = selectedRecords.join('\n\n');
  const fileName = `扫码记录_${formatTime().replace(/[: ]/g, '-')}.txt`;

  // 使用 uni.getFileSystemManager 创建临时文件
  const fs = uni.getFileSystemManager();
  const tempFilePath = `${uni.env.USER_DATA_PATH}/${fileName}`;

  fs.writeFile({
    filePath: tempFilePath,
    data: content,
    encoding: 'utf8',
    success: () => {
      uni.saveFile({
        tempFilePath: tempFilePath,
        success: (res) => {
          uni.showToast({
            title: '导出成功',
            icon: 'success',
            complete: () => {
              exportMode.value = false;
              scanRecords.value.forEach(r => r.selected = false);
            }
          });
        },
        fail: (err) => {
          uni.showToast({
            title: '导出失败',
            icon: 'none',
            complete: () => {
              exportMode.value = false;
              scanRecords.value.forEach(r => r.selected = false);
            }
          });
        }
      });
    },
    fail: (err) => {
      uni.showToast({
        title: '写入文件失败',
        icon: 'none',
        complete: () => {
          exportMode.value = false;
          scanRecords.value.forEach(r => r.selected = false);
        }
      });
    }
  });
};

// 核验功能
const verifyRecords = () => {
  const outboundContents = scanRecords.value.map(r => r.content);
  const inboundContents = inboundRecords.value.map(r => r.content);
  unmatchedOutbound.value = outboundContents.filter(content =>!inboundContents.includes(content));
  unmatchedInbound.value = inboundContents.filter(content =>!outboundContents.includes(content));
  matchedCount.value = inboundRecords.value.filter(record => outboundContents.includes(record.content)).length;

  if (unmatchedOutbound.value.length === 0 && unmatchedInbound.value.length === 0) {
    verificationResult.value = `核验通过，所有出库和入库记录标识一致，核验通过的箱数：${matchedCount.value}。`;
  } else {
    let message = '核验未通过，以下是不匹配的记录：\n';
    if (unmatchedOutbound.value.length > 0) {
      message += '未入库的出库记录：\n';
      message += unmatchedOutbound.value.map(content => `- ${content}`).join('\n');
      message += '\n';
    }
    if (unmatchedInbound.value.length > 0) {
      message += '未出库的入库记录：\n';
      message += unmatchedInbound.value.map(content => `- ${content}`).join('\n');
    }
    message += `\n核验通过的箱数算法：出入库完全匹配记录之和`;
    verificationResult.value = message;
  }
};

// 出库核验功能
const verifyOutbound = () => {
  uni.showModal({
    title: '预计入库箱数',
    content: '请输入预计入库的箱数',
    editable: true,
    success: (res) => {
      if (res.confirm && res.content) {
        const maxCount = parseInt(res.content);
        const outboundNumbers = scanRecords.value.map(record => {
          const match = record.content.match(/^(\d+)-/);
          return match? parseInt(match[1]) : null;
        }).filter(num => num!== null);
        const allNumbers = Array.from({ length: maxCount }, (_, i) => i + 1);
        outboundMissingNumbers.value = allNumbers.filter(num =>!outboundNumbers.includes(num));
        if (outboundMissingNumbers.value.length === 0) {
          outboundVerificationResult.value = '出库记录编号连续，核验通过。';
        } else {
          outboundVerificationResult.value = `缺失的出库编号：${outboundMissingNumbers.value.join(', ')}`;
        }
      }
    }
  });
};

// 入库核验功能
const verifyInbound = () => {
  uni.showModal({
    title: '预计出库箱数',
    content: '请输入预计出库的箱数',
    editable: true,
    success: (res) => {
      if (res.confirm && res.content) {
        const maxCount = parseInt(res.content);
        const inboundNumbers = inboundRecords.value.map(record => {
          const match = record.content.match(/^(\d+)-/);
          return match? parseInt(match[1]) : null;
        }).filter(num => num!== null);
        const allNumbers = Array.from({ length: maxCount }, (_, i) => i + 1);
        inboundMissingNumbers.value = allNumbers.filter(num =>!inboundNumbers.includes(num));
        if (inboundMissingNumbers.value.length === 0) {
          inboundVerificationResult.value = '入库记录编号连续，核验通过。';
        } else {
          inboundVerificationResult.value = `缺失的入库编号：${inboundMissingNumbers.value.join(', ')}`;
        }
      }
    },
    // 新增：输入框获得焦点时清除提示文本
    complete: (res) => {
      if (res.errMsg === 'showModal:ok' && res.confirm) {
        const input = document.querySelector('.uni-modal-input');
        if (input) {
          input.addEventListener('focus', () => {
            input.placeholder = '';
          });
        }
      }
    }
  });
};

// 生成随机颜色
const getRandomColor = (index) => {
  const colors = ['#FF5733', '#5733FF'];
  return colors[index % colors.length];
};

// 新增：应用筛选
const applyFilter = () => {
  if (isFiltering.value) {
    // 取消筛选
    filterKeyword.value = '';
    if (page.value === 'outbound') {
      scanRecords.value = originalOutboundRecords.value;
    } else if (page.value === 'inbound') {
      inboundRecords.value = originalInboundRecords.value;
    }
    isFiltering.value = false;
  } else {
    // 应用筛选
    if (page.value === 'outbound') {
      scanRecords.value = filteredOutboundRecords.value;
    } else if (page.value === 'inbound') {
      inboundRecords.value = filteredInboundRecords.value;
    }
    isFiltering.value = true;
  }
};
</script>

<style lang="scss" scoped>
.main-container {
  height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: #f8f8f8;
}

.main-page {
  text-align: center;
}

.title {
  font-size: 16px;
  font-weight: bold;
  margin-bottom: 25px;
}

.buttons {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.main-button {
  padding: 15px 30px;
  font-size: 15px;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, #007AFF, #0051ff);
  border: none;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 122, 255, 0.2);
  transition: all 0.3s ease;
  cursor: pointer;
  width: 218px;
}

.main-button:hover {
  background: linear-gradient(135deg, #0051ff, #003a99);
  box-shadow: 0 6px 12px rgba(0, 122, 255, 0.3);
  transform: translateY(-2px);
}

.main-button:active {
  transform: translateY(1px);
  box-shadow: 0 2px 4px rgba(0, 122, 255, 0.2);
}

.container {
  height: 100vh;
  background: #f8f8f8;
}

.header-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: white;
  box-shadow: 0 2px 12px rgba(0,0,0,0.08);
  position: sticky;
  top: 0;
  z-index: 100;
}

.scan-btn {
  width: 44px;
  height: 44px;
  border-radius: 8px;
  background: linear-gradient(135deg, #007AFF, #0051ff);
  display: flex;
  vertical-align: middle; /*垂直居中*/
  align-items: center; /*水平居中*/  
  justify-content: center;
  box-shadow: 0 4px 8px rgba(0,122,255,0.2);
  
  &:active {
    opacity: 0.9;
    transform: scale(0.98);
  }
}

.scan-icon {
  color: white;
  font-size: 28px;
  font-weight: bold;
}

.scan-count {
  font-size: 15px;
  color: #333;
  font-weight: 500;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}



.export-btn {
  padding: 6px 14px;
  background: #f0f0f0;
  border-radius: 6px;
  font-size: 14px;
  color: #007AFF;
  
  &.disabled {
    opacity: 0.5;
    pointer-events: none;
  }
  
  &:active {
    background: #e0e0e0;
  }
}


.cancel-btn {
  padding: 6px 14px;
  background: #ff4444;
  color: white;
  border-radius: 6px;
  font-size: 14px;
}

.record-list {
  height: calc(100vh - 68px);
  padding: 12px;
}

.record-item {
  display: flex;
  align-items: center;
  padding: 16px;
  margin-bottom: 25px;
  background: white;
  border-radius: 10px;
  box-shadow: 0 2px 6px rgba(0,0,0,0.05);
  
  &.highlight {
    background: #fffbe6;
    animation: highlight 3s;
  }
}

@keyframes highlight {
  0% { background: #fffbe6; }
  100% { background: white; }
}

.checkbox {
  width: 20px;
  height: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-right: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
}



.check-icon {
  color: #007AFF;
  font-size: 14px;
}

.content {
  flex: 1;
  color: #333;
  font-size: 15px;
  margin-right: 15px;
  word-break: break-all;
}

.action-btns {
  display: flex;
  gap: 20px;
  
  text {
    padding: 4px;
    font-size: 18px;
  }
}

.edit-btn {
  color: #666;
}

.delete-btn {
  color: #ff4444;
}

.modal-mask {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: white;
  width: 80%;
  padding: 24px;
  border-radius: 12px;
}

.password-input {
  height: 44px;
  padding: 0 12px;
  border: 1px solid #eee;
  border-radius: 8px;
  margin-bottom: 20px;
}

.modal-btns {
  display: flex;
  justify-content: flex-end;
  gap: 20px;
  
  text {
    padding: 8px 20px;
    border-radius: 6px;
    font-size: 14px;
  }
}

.confirm-btn {
  background: #007AFF;
  color: white;
}

.empty-tip {
  text-align: center;
  color: #999;
  padding: 40px 0;
  font-size: 14px;
}

.back-btn {
            position: fixed;
            bottom: 1px;
            left: 50%;
            transform: translateX(-50%);
            padding: 10px 30px;
            background: #007AFF;
            color: white;
            border-radius: 6px;
            font-size: 15px;
            max-width: 600px;
            width: 99%;
            display: flex;
            justify-content: center;
            align-items: center;
}



.result {
  padding: 20px;
  text-align: center;
  font-size: 14px;
}

.verify-button {
  padding: 8px 16px;
  font-size: 14px;
  font-weight: 600;
  color: whitesmoke;
  background: linear-gradient(135deg, #55aaff, #00aaff);
  border: none;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 122, 255, 0.2);
  transition: all 0.3s ease;
  cursor: pointer;
   max-width: 120px;
   display: inline-block;
  
	  margin: 0 8px; /* 按钮之间的间距 */
	 padding: 5px 15px; /* 按钮内边距 */
	
}

.verify-button:hover {
  background: linear-gradient(135deg, #0051ff, #0040a8);
  box-shadow: 0 6px 12px rgba(0, 122, 255, 0.3);
  transform: translateY(-2px);
}

.verify-button:active {
  transform: translateY(1px);
  box-shadow: 0 2px 4px rgba(0, 122, 255, 0.2);
}


.divider {
  height: 1px;
  background-color: #ccc;
  margin: 8px 0;
}



.matched-count {
  margin-top: 20px;
  text-align: center;
}



.matched-count-text {
  font-size: 15px;
  color: #333;
  margin-left: 10px;
}

.alternate-bg {
  background-color: #a2a2a2;
}
</style>    