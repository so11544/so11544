<template>
  <view class="container">
    <view class="login-box">
      <image class="logo" src="/static/72.png"></image>
      <view class="form-item">
        <view class="input-group">
          <input
            v-model="username"
            class="input"
            placeholder="用户名"
            placeholder-style="color: #bdbdbd;font-size: .9375rem"
          />
          <picker :range="usernameList" @change="handleUsernameChange">
            <view class="picker-trigger">
              <i class="fa-solid fa-caret-down"></i>
            </view>
          </picker>
        </view>
      </view>
      <view class="form-item">
        <input
          v-model="password"
          class="input"
          type="password"
          placeholder="密码"
          placeholder-style="color: #bdbdbd;font-size: .9375rem"
        />
      </view>
      <view class="form-item remember-me">
        <checkbox :checked="rememberMe" @change="handleCheckboxChange">记住用户名和密码</checkbox>
      </view>
      <button
        :disabled="loading"
        @click="handleLogin"
        class="login-btn"
      >
        {{ loading ? '登录中...' : '登录' }}
      </button>
      <view class="server-time" :style="{ color: serverTimeError ? 'red' : 'inherit' }">
        {{ serverTimeError ? '服务器连接失败或网络异常' : `ServerTime:${serverTime}` }}
      <view class="extra-text"><br>
              version 2.0.0
            </view>
	  </view>
    </view>
  </view>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const serverUrl = 'http://123.12.42.171:3000/login';
const serverTimeUrl = 'http://123.12.42.171:3001/test';
const USER_AGENT = 'StorageClient/1.0.0';

const username = ref('');
const password = ref('');
const loading = ref(false);
const rememberMe = ref(true);
const serverTime = ref('');
const serverTimeError = ref(false);
const retryCount = ref(0);
const maxRetries = 20;
let timer = null;
const usernameList = ref([]);
const userPasswordMap = ref({});

const handleUsernameChange = (e) => {
  const selectedIndex = e.mp.detail.value;
  const selectedUsername = usernameList.value[selectedIndex];
  username.value = selectedUsername;
  password.value = userPasswordMap.value[selectedUsername] || '';
};

const handleCheckboxChange = (e) => {
  rememberMe.value = e.mp.detail.value[0] === 'on';
};

const validateForm = () => {
  if (!username.value.trim()) {
    uni.showToast({ title: '用户名不能为空', icon: 'none' });
    return false;
  }
  if (!password.value.trim()) {
    uni.showToast({ title: '密码不能为空', icon: 'none' });
    return false;
  }
  return true;
};

const fetchServerTime = async () => {
  try {
    const { data, statusCode } = await uni.request({
      url: serverTimeUrl,
      method: 'GET',
      timeout: 10000,
      header: {
        'Content-Type': 'application/json',
        'User-Agent': USER_AGENT
      }
    });
    if (statusCode === 200) {
      serverTime.value = data.serverTime;
      serverTimeError.value = false;
      retryCount.value = 0;
    } else {
      throw new Error('请求失败');
    }
  } catch (error) {
    console.error('获取服务器时间异常:', error);
    serverTimeError.value = true;
    retryCount.value++;
    if (retryCount.value >= maxRetries) {
      clearInterval(timer);
    }
  }
};

onMounted(() => {
  const storedUsername = uni.getStorageSync('rememberedUsername');
  const storedPassword = uni.getStorageSync('rememberedPassword');
  if (storedUsername && storedPassword) {
    username.value = storedUsername;
    password.value = storedPassword;
    rememberMe.value = true;
    if (!usernameList.value.includes(storedUsername)) {
      usernameList.value.push(storedUsername);
    }
    userPasswordMap.value[storedUsername] = storedPassword;
  }
  fetchServerTime();
  timer = setInterval(fetchServerTime, 3000);
});

onUnmounted(() => {
  clearInterval(timer);
});

const handleLogin = async () => {
  if (!validateForm()) return;
  loading.value = true;

  if (rememberMe.value) {
    uni.setStorageSync('rememberedUsername', username.value);
    uni.setStorageSync('rememberedPassword', password.value);
    if (!usernameList.value.includes(username.value)) {
      usernameList.value.push(username.value);
    }
    userPasswordMap.value[username.value] = password.value;
  } else {
    uni.removeStorageSync('rememberedUsername');
    uni.removeStorageSync('rememberedPassword');
    delete userPasswordMap.value[username.value];
  }

  try {
    const { data, statusCode } = await uni.request({
      url: serverUrl,
      method: 'POST',
      timeout: 10000,
      header: {
        'Content-Type': 'application/json',
        'User-Agent': USER_AGENT
      },
      data: {
        username: username.value,
        password: password.value
      }
    });

    if (statusCode === 200) {
      handleLoginSuccess(data);
    } else if (statusCode === 400) {
      uni.showToast({ title: data.error, icon: 'none' });
    } else if (statusCode === 401) {
      uni.showToast({ title: data.error, icon: 'none' });
    } else {
      uni.showToast({ title: '登录失败，请稍后重试', icon: 'none' });
    }
  } catch (error) {
    console.error('请求异常:', error);
    if (error.errMsg === 'request:fail timeout') {
      uni.showToast({ title: '请求超时，请稍后重试', icon: 'none' });
    } else {
      uni.showToast({ title: '网络连接失败', icon: 'none' });
    }
  } finally {
    loading.value = false;
  }
};

const handleLoginSuccess = (responseData) => {
  uni.setStorageSync('userInfo', {
    username: username.value,
    telephone: responseData.telephone_number,
    lastLogin: responseData.lastLogin
  });

  uni.setStorageSync('currentUsername', username.value);

  uni.navigateTo({
    url: '/pages/index/index',
    success: () => {
      uni.showToast({ title: '登录成功', icon: 'success' });
    },
    fail: (err) => {
      console.error('跳转失败:', err);
    }
  });
};
</script>

<style scoped lang="scss">
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background: linear-gradient(135deg, #f5f7fa, #c3cfe2);
}

.login-box {
  width: 85%;
  max-width: 400px;
  padding: 1.25rem;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 0.75rem;
  box-shadow: 0 0.25rem 0.75rem rgba(0, 0, 0, 0.1);
}

.logo {
  width: 6.25rem;
  height: 6.25rem;
  margin: 0 auto 1.875rem;
  display: block;
}

.form-item {
  margin-bottom: 1.5rem;
}

.input-group {
  position: relative;
  display: flex;
  align-items: center;
}

.input {
  height: 3rem;
  padding: 0 1rem;
  border: 0.0625rem solid #e0e0e0;
  border-radius: 0.375rem;
  font-size: 1rem;
  transition: border-color 0.3s;
  flex: 1;
  box-sizing: border-box;
}

.picker-trigger {
  position: absolute;
  top: 0;
  right: 0;
  height: 3rem;
  width: 3rem;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  z-index: 1;
}

.login-btn {
  background: #007aff;
  color: #fff;
  height: 3rem;
  line-height: 3rem;
  border-radius: 0.375rem;
  font-size: 1.0625rem;
  letter-spacing: 0.0625rem;
  transition: opacity 0.3s;

  &[disabled] {
    background: #c0c4cc;
    opacity: 0.7;
  }
}

.remember-me {
  display: flex;
  align-items: center;
  margin-bottom: 1rem;

  checkbox {
    margin-right: 1rem;
  }
}

.server-time {
  margin-top: 1rem;
  text-align: center;
  font-size: 0.875rem;
}
.extra-text {
            font-size: 12px;
        }
</style>    