<script setup>
import githubIcon from '@/assets/img/github.png'
import qqIcon from '@/assets/img/QQ.png'
import wechatIcon from '@/assets/img/wechat.png'
import mailIcon from '@/assets/img/mail.png'
import { ref, onMounted } from 'vue'
import EmailModal from '@/components/EmailModal.vue'

const showEmailModal = ref(false)

const links = [
  { name: 'Github', icon: githubIcon, color: '#333', url: 'https://github.com/Dls-git/myblog' },
  { name: 'QQ', icon: qqIcon, color: '#12B7F5', url: 'https://im.qq.com/index.shtml' },
  { name: 'WeChat', icon: wechatIcon, color: '#07C160', url: 'https://weixin.qq.com/' },
  { name: 'Email', icon: mailIcon, color: '#EA4335', action: 'modal' }
]

const handleLinkClick = (link) => {
  if (link.action === 'modal') {
    showEmailModal.value = true
  } else if (link.url) {
    window.open(link.url, '_blank')
  }
}

// 打字机效果
const fullText = "Hi new friend, I'm 月鸾随👋\nWelcome to my blog!"
const displayedText = ref('')
const typingSpeed = 100 // 打字速度 ms

onMounted(() => {
  let i = 0
  const typeWriter = () => {
    if (i < fullText.length) {
      displayedText.value += fullText.charAt(i)
      i++
      setTimeout(typeWriter, typingSpeed)
    }
  }
  typeWriter()
})
</script>

<template>
  <div class="container">
    <div class="text">
      <h2 class="text-title">
        <span class="typewriter-text">{{ displayedText }}</span>
        <span class="cursor">|</span>
      </h2>
      <p class="text-content small-font">What does not kill me, makes me stronger</p>

      <div class="other-link">
        <div
          v-for="(link, index) in links"
          :key="index"
          class="link-item"
          @click="handleLinkClick(link)"
        >
          <div class="icon" :style="{ '--hover-color': link.color }">
            <img :src="link.icon" :alt="link.name">
          </div>
          <span class="link-text">{{ link.name }}</span>
        </div>
      </div>
    </div>
    <div class="my-image ">
      <img src="@/assets/img/head.png" alt="黄业添">
    </div>

    <!-- 邮箱弹窗 -->
    <EmailModal
      :visible="showEmailModal"
      @close="showEmailModal = false"
    />
  </div>
</template>

<style scoped lang="scss">
/* ... (保留原有其他样式) ... */

.text-title {
  font-size: 48px;
  font-weight: bold;
  color: rgb(var(--color-text-primary));
  line-height: 1.2;
  min-height: 120px; /* 防止打字时高度跳动 */
  white-space: pre-wrap; /* 保持换行 */
}

/* 光标闪烁动画 */
.cursor {
  display: inline-block;
  margin-left: 2px;
  animation: blink 1s step-end infinite;
  color: rgb(var(--color-text-primary));
  font-weight: 100;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0; }
}

/* ... (保留 container, text, other-link, my-image 等样式) ... */
.container{
  display: flex;
  padding-top: 150px; /* 调整顶部间距 */
  min-height: 600px;
  max-width: 1200px;
  margin: 0 auto;
  align-items: center;
}

.text{
  flex: 1;
  padding-right: 50px;
  /* border: rebeccapurple 5px solid; */ /* 移除调试边框 */

  .text-content{
    margin-top: 20px;
    font-size: 20px;
    color: rgb(var(--color-text-primary));
    opacity: 0.7;
  }
}

.other-link {
  margin-top: 50px;
  display: flex;
  gap: 20px; /* 按钮之间的间距 */
}

/* 手风琴按钮样式 */
.link-item {
  display: flex;
  align-items: center;
  width: 60px; /* 初始宽度 */
  height: 60px;
  background: rgb(var(--color-bg-secondary));
  border-radius: 30px;
  overflow: hidden;
  transition: all 0.4s ease-out;
  cursor: pointer;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);

  &:hover {
    width: 160px; /* 展开后的宽度 */
    background: white; /* 展开后背景变白 */

    .icon {
      background: var(--hover-color); /* 悬停时图标背景变色 */

      img {
        filter: brightness(0) invert(1); /* 悬停时图标变白 (假设是黑色或彩色图标) */
      }
    }

    .link-text {
      opacity: 1;
    }
  }
}

/* 图标容器 */
.icon {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  display: flex;
  justify-content: center;
  align-items: center;
  transition: all 0.4s ease-out;
  flex-shrink: 0; /* 防止图标被压缩 */

  img {
    width: 28px;
    height: 28px;
    object-fit: contain;
    transition: all 0.3s;
  }
}

/* 文字样式 */
.link-text {
  opacity: 0;
  margin-left: 10px;
  font-size: 18px;
  font-weight: 600;
  color: #333;
  white-space: nowrap;
  transition: opacity 0.3s 0.1s; /* 稍微延迟显示文字 */
}

.my-image{
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  /* border: blue 5px solid; */ /* 移除调试边框 */

  img{
    border-radius: 50%;
    height: 400px;
    width: 400px;
    object-fit: cover;
    box-shadow: 0 10px 30px rgba(0,0,0,0.2);
    transition: transform 0.3s ease;

    &:hover {
      transform: scale(1.05);
    }
  }
}
</style>
