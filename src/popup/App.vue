<template>
  <div class="content">
    <popup-item
      :title="item.title"
      :isShow="item.isShow"
      :handlerType="item.handlerType"
      :icon="item.icon"
      @handler="cardHandler"
      v-for="item in popupItems"
      :key="item.handlerType"
    ></popup-item>

    <block-popover-popup-item
      @handler="appendColorBlock"
      :items="colorBlockItems"
      title="插入提示框"
      icon="el-icon-potato-strips"
      itemWidth="80"
      itemHeight="45"
    ></block-popover-popup-item>

    <block-popover-popup-item
      @handler="appendColorHeader"
      :items="colorHeaderItems"
      title="插入彩色标题头"
      icon="el-icon-ice-cream-round"
      itemWidth="80"
      itemHeight="45"
    ></block-popover-popup-item>

    <block-popover-popup-item @handler="appendImg" :items="imgItems" title="插入表情" icon="el-icon-grape"></block-popover-popup-item>
  </div>
</template>

<script>
import BlockPopoverPopupItem from '../componts/BlockPopoverPopupItem.vue';
import PopupItem from '../componts/PopupItem.vue';
import { mood, colorBlockItems, colorHeaderItems, popupItems } from '../common/config.js';
import { createDoc, readDoc } from '../common/yuque-sdk';

var dayjs = require('dayjs');

export default {
  components: { PopupItem, BlockPopoverPopupItem },
  data() {
    return {
      popupItems: popupItems,
      imgItems: mood,
      colorBlockItems: colorBlockItems,
      colorHeaderItems: colorHeaderItems,
    };
  },
  methods: {
    // 卡片处理
    cardHandler(handlerType) {
      switch (handlerType) {
        case 'markdown':
          this.openMarkdown();
          break;
        case 'html':
          this.openHtml();
          break;
        case 'copyUrl':
          this.copyUrl();
          break;
        case 'toc':
          this.sendSimpleMessageToContentScript({ cmd: 'toc', value: '' });
          break;
        case 'prepend_img':
          this.sendSimpleMessageToContentScript({ cmd: 'prepend_img', value: '' });
          break;
        case 'prepend_blank':
          this.sendSimpleMessageToContentScript({ cmd: 'prepend_blank', value: '' });
          break;
        case 'format_content':
          this.sendSimpleMessageToContentScript({ cmd: 'format_content', value: '' });
          break;
        case 'get_books':
          this.getBooks();
          break;
        case 'markmap':
          this.openMarkmap();
          break;
        case 'setting':
          this.openUrl('/options/options.html');
          break;
        case 'help':
          this.openUrl('https://www.yuque.com/yag8nv');
          break;
        case 'generator_header':
          this.sendSimpleMessageToContentScript({ cmd: 'generator_header', value: '' });
          break;
        case 'clipper':
          this.openEditor();
          break;
        case 'diary':
          this.openDiary();
          break;
        default:
          break;
      }
    },
    // 以 markdown 打开
    openMarkdown() {
      this.getCurrentTab(tab => {
        window.open(this.urlFormat(tab.url) + '/markdown?plain=true&linebreak=false&anchor=false', '_blank');
      });
    },
    // 以 html 打开
    openHtml() {
      this.getCurrentTab(tab => {
        window.open(this.urlFormat(tab.url) + '/html', '_blank');
      });
    },
    // 格式化 url，如果是语雀的链接就去除 /edit
    urlFormat(url) {
      if (url.includes('yuque')) {
        return url.replace('/edit', '');
      } else {
        return url;
      }
    },
    // 添加表情
    appendImg(imgUrl) {
      this.sendMessageToContentScript({ cmd: 'append_img', value: imgUrl }, function(response) {
        console.log('来自content的回复：' + response);
      });
    },
    // 打开 markmap 目录
    openMarkmap() {
      this.getCurrentTab(tab => {
        if (!tab.url.includes('markdown')) {
          this.sendMessageToContentScript({ cmd: 'clipper' }, function(response) {
            chrome.storage.local.set({ temp: response.content }, () => {
              window.open('/markmap/markmap.html', '_blank');
            });
          });
        } else {
          this.sendMessageToContentScript({ cmd: 'get_markmap' }, function(response) {
            console.log(response);
            chrome.storage.local.set({ temp: response }, () => {
              window.open('/markmap/markmap.html', '_blank');
            });
          });
        }
      });
    },
    // 打开编辑框
    openEditor() {
      this.sendMessageToContentScript({ cmd: 'clipper' }, function(response) {
        chrome.storage.local.set({ tempContent: response.content, tempTitle: response.title }, () => {
          window.open('/editor/editor.html', '_blank');
        });
      });
    },
    // 打开页面
    openUrl(url) {
      window.open(url, '_blank');
    },
    // 添加提示框
    appendColorBlock(imgUrl) {
      this.sendMessageToContentScript({ cmd: 'append_color_block', value: imgUrl }, function(response) {
        console.log('来自content的回复：' + response);
      });
    },
    appendColorHeader(imgUrl) {
      this.sendMessageToContentScript({ cmd: 'append_color_header', value: imgUrl }, function(response) {
        console.log('来自content的回复：' + response);
      });
    },
    isArticleUrl() {
      this.getCurrentTab(tab => {
        currentTabIsArticleUrl = /https:\/\/.*\.yuque\.com\/.+\/.+\/.+/.test(tab.url);
      });
    },
    copy(text) {
      let transfer = document.createElement('input');
      document.body.appendChild(transfer);
      transfer.value = text; // 这里表示想要复制的内容
      transfer.focus();
      transfer.select();
      if (document.execCommand('copy')) {
        document.execCommand('copy');
      }
      transfer.blur();
      console.log('复制成功');
      document.body.removeChild(transfer);
    },
    // 获得书籍目标
    getBooks() {
      this.sendMessageToContentScript({ cmd: 'get_books', value: '' }, response => {
        this.copy(response);
        if (response) {
          this.notify('复制成功', '图书目录');
        } else {
          this.notify('复制失败', '请在目录面使用此功能');
        }
      });
    },
    openDiary() {
      chrome.storage.sync.get({ yuqueOption: {} }, res => {
        let config = res.yuqueOption;
        if (!config || !config.yuqueToken || !config.yuqueRepo || !config.yuqueUsername) {
          chrome.notifications.create(null, {
            type: 'basic',
            iconUrl: '/icons/icon_48.png',
            title: '打开失败',
            message: '请先配置 yuque 信息',
          });
        } else {
          let today = dayjs().format('YYYYMMDD');
          // 获得当天的文章
          readDoc({
            slug: today,
            token: config.yuqueToken,
            repo: config.yuqueRepo,
          })
            .then(res => {
              this.openUrl(`https://www.yuque.com/${config.yuqueRepo}/${today}/edit`);
            })
            .catch(res => {
              createDoc({
                title: dayjs().format('YYYY-MM-DD'),
                content: '',
                slug: today,
                token: config.yuqueToken,
                repo: config.yuqueRepo,
              })
                .then(res => {
                  this.openUrl(`https://www.yuque.com/${config.yuqueRepo}/${today}/edit`);
                })
                .catch(err => {
                  chrome.notifications.create(null, {
                    type: 'basic',
                    iconUrl: '/icons/icon_48.png',
                    title: '日记生成失败',
                    message: '原因：' + err.response.data.message,
                  });
                });
            });
        }
      });
    },
    copyUrl() {
      this.getCurrentTab(tab => {
        let urlByMarkdown = '[' + tab.title + '](' + this.urlFormat(tab.url) + ')';
        this.copy(urlByMarkdown);
        // 通知
        this.notify('复制成功', '复制内容： ' + urlByMarkdown);
      });
    },
    getCurrentTab(callback) {
      chrome.tabs.query(
        {
          active: true,
          currentWindow: true,
        },
        function(tabs) {
          if (callback) callback(tabs.length ? tabs[0] : null);
        }
      );
    },
    // 通知
    notify(title, message) {
      chrome.notifications.create(null, {
        type: 'basic',
        iconUrl: '/icons/icon_48.png',
        title: title,
        message: message,
      });
    },
    // 发送简单消息给 content script
    sendSimpleMessageToContentScript(request) {
      this.sendMessageToContentScript(request, function(response) {});
    },
    // 发送消息给 content script
    sendMessageToContentScript(message, callback) {
      chrome.tabs.query({ active: true, currentWindow: true }, function(tabs) {
        chrome.tabs.sendMessage(tabs[0].id, message, function(response) {
          if (callback) callback(response);
        });
      });
    },
  },
};
</script>

<style scoped>
.content {
  padding: 6px 1px 6px 1px;
  width: 200px;
  background-color: #f3f4f7;
  border-radius: 5px;
}
</style>
