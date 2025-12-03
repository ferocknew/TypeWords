<script setup lang="ts">
import { nextTick, onMounted, ref, watch } from "vue";
import { useSettingStore } from "@/stores/setting.ts";
import { getAudioFileUrl, usePlayAudio } from "@/hooks/sound.ts";
import { getShortcutKey, useEventListener } from "@/hooks/event.ts";
import { checkAndUpgradeSaveDict, checkAndUpgradeSaveSetting, cloneDeep, loadJsLib, shakeCommonDict } from "@/utils";
import { DefaultShortcutKeyMap, ShortcutKey, WordPracticeMode } from "@/types/types.ts";
import BaseButton from "@/components/BaseButton.vue";
import VolumeIcon from "@/components/icon/VolumeIcon.vue";
import { useBaseStore } from "@/stores/base.ts";
import { saveAs } from "file-saver";
import {
  APP_NAME, APP_VERSION, EMAIL,
  EXPORT_DATA_KEY, GITHUB, Host,
  LOCAL_FILE_KEY,
  Origin,
  PracticeSaveArticleKey,
  PracticeSaveWordKey, SAVE_DICT_KEY, SAVE_SETTING_KEY, SoundFileOptions
} from "@/config/env.ts";
import dayjs from "dayjs";
import BasePage from "@/components/BasePage.vue";
import Toast from '@/components/base/toast/Toast.ts'
import { Option, Select } from "@/components/base/select";
import Switch from "@/components/base/Switch.vue";
import Slider from "@/components/base/Slider.vue";
import RadioGroup from "@/components/base/radio/RadioGroup.vue";
import Radio from "@/components/base/radio/Radio.vue";
import InputNumber from "@/components/base/InputNumber.vue";
import PopConfirm from "@/components/PopConfirm.vue";
import Textarea from "@/components/base/Textarea.vue";
import SettingItem from "@/pages/setting/SettingItem.vue";
import { get, set } from "idb-keyval";
import { useRuntimeStore } from "@/stores/runtime.ts";
import { useUserStore } from "@/stores/user.ts";
import { useExport } from "@/hooks/export.ts";
import MigrateDialog from "@/components/MigrateDialog.vue";

const emit = defineEmits<{
  toggleDisabledDialogEscKey: [val: boolean]
}>()

const tabIndex = $ref(3)
const settingStore = useSettingStore()
const runtimeStore = useRuntimeStore()
const store = useBaseStore()

//@ts-ignore
const gitLastCommitHash = ref(LATEST_COMMIT_HASH);
const simpleWords = $computed({
  get: () => store.simpleWords.join(','),
  set: v => {
    try {
      store.simpleWords = v.split(',');
    } catch (e) {

    }
  }
})

let editShortcutKey = $ref('')

const disabledDefaultKeyboardEvent = $computed(() => {
  return editShortcutKey && tabIndex === 3
})

watch(() => disabledDefaultKeyboardEvent, v => {
  emit('toggleDisabledDialogEscKey', !!v)
})

// 监听编辑快捷键状态变化，自动聚焦输入框
watch(() => editShortcutKey, (newVal) => {
  if (newVal) {
    // 使用nextTick确保DOM已更新
    nextTick(() => {
      focusShortcutInput()
    })
  }
})

useEventListener('keydown', (e: KeyboardEvent) => {
  if (!disabledDefaultKeyboardEvent) return

  // 确保阻止浏览器默认行为
  e.preventDefault()
  e.stopPropagation()

  let shortcutKey = getShortcutKey(e)
  // console.log('e', e, e.keyCode, e.ctrlKey, e.altKey, e.shiftKey)
  // console.log('key', shortcutKey)

  // if (shortcutKey[shortcutKey.length-1] === '+') {
  //   settingStore.shortcutKeyMap[editShortcutKey] = DefaultShortcutKeyMap[editShortcutKey]
  //   return ElMessage.warning('设备失败！')
  // }

  if (editShortcutKey) {
    if (shortcutKey === 'Delete') {
      settingStore.shortcutKeyMap[editShortcutKey] = ''
    } else {
      // 忽略单独的修饰键
      if (shortcutKey === 'Ctrl+' || shortcutKey === 'Alt+' || shortcutKey === 'Shift+' ||
          e.key === 'Control' || e.key === 'Alt' || e.key === 'Shift') {
        return;
      }

      for (const [k, v] of Object.entries(settingStore.shortcutKeyMap)) {
        if (v === shortcutKey && k !== editShortcutKey) {
          settingStore.shortcutKeyMap[editShortcutKey] = DefaultShortcutKeyMap[editShortcutKey]
          return Toast.warning('快捷键重复！')
        }
      }
      settingStore.shortcutKeyMap[editShortcutKey] = shortcutKey
    }
  }
})

function handleInputBlur() {
  // 输入框失焦时结束编辑状态
  editShortcutKey = ''
}

function handleBodyClick() {
  if (editShortcutKey) {
    editShortcutKey = ''
  }
}

function focusShortcutInput() {
  // 找到当前正在编辑的快捷键输入框
  const inputElements = document.querySelectorAll('.set-key input')
  if (inputElements && inputElements.length > 0) {
    // 聚焦第一个找到的输入框
    const inputElement = inputElements[0] as HTMLInputElement
    inputElement.focus()
  }
}

// 快捷键中文名称映射
function getShortcutKeyName(key: string): string {
  const shortcutKeyNameMap = {
    'ShowWord': '显示单词',
    'EditArticle': '编辑文章',
    'Next': '下一个',
    'Previous': '上一个',
    'ToggleSimple': '切换已掌握状态',
    'ToggleCollect': '切换收藏状态',
    'NextChapter': '下一组',
    'PreviousChapter': '上一组',
    'RepeatChapter': '重复本组',
    'DictationChapter': '默写本组',
    'PlayWordPronunciation': '播放发音',
    'ToggleShowTranslate': '切换显示翻译',
    'ToggleDictation': '切换默写模式',
    'ToggleTheme': '切换主题',
    'ToggleConciseMode': '切换简洁模式',
    'TogglePanel': '切换面板',
    'RandomWrite': '随机默写',
    'NextRandomWrite': '继续随机默写',
    'KnowWord': '认识单词',
    'UnknownWord': '不认识单词',
  }

  return shortcutKeyNameMap[key] || key
}

function resetShortcutKeyMap() {
  editShortcutKey = ''
  settingStore.shortcutKeyMap = cloneDeep(DefaultShortcutKeyMap)
  Toast.success('恢复成功')
}

let importLoading = $ref(false)

const {loading: exportLoading, exportData} = useExport()

function importJson(str: string, notice: boolean = true) {
  let obj = {
    version: -1,
    val: {
      setting: {},
      dict: {},
      [PracticeSaveWordKey.key]: {},
      [PracticeSaveArticleKey.key]: {},
      [APP_VERSION.key]: {},
    }
  }
  try {
    obj = JSON.parse(str)
    let data = obj.val
    let settingState = checkAndUpgradeSaveSetting(data.setting)
    settingState.load = true
    settingStore.setState(settingState)
    let baseState = checkAndUpgradeSaveDict(data.dict)
    baseState.load = true
    store.setState(baseState)
    if (obj.version >= 3) {
      try {
        let save: any = obj.val[PracticeSaveWordKey.key] || {}
        if (save.val && Object.keys(save.val).length > 0) {
          localStorage.setItem(PracticeSaveWordKey.key, JSON.stringify(obj.val[PracticeSaveWordKey.key]))
        }
      } catch (e) {
        //todo 上报
      }
    }
    if (obj.version >= 4) {
      try {
        let save: any = obj.val[PracticeSaveArticleKey.key] || {}
        if (save.val && Object.keys(save.val).length > 0) {
          localStorage.setItem(PracticeSaveArticleKey.key, JSON.stringify(obj.val[PracticeSaveArticleKey.key]))
        }
      } catch (e) {
        //todo 上报
      }
      try {
        let r: any = obj.val[APP_VERSION.key] || -1
        set(APP_VERSION.key, r)
        runtimeStore.isNew = r ? (APP_VERSION.version > Number(r)) : true
      } catch (e) {
        //todo 上报
      }
    }
    notice && Toast.success('导入成功！')
  } catch (err) {
    return Toast.error('导入失败！')
  }
}

async function importData(e) {
  let file = e.target.files[0]
  if (!file) return
  if (file.name.endsWith(".json")) {
    let reader = new FileReader();
    reader.onload = function (v) {
      let str: any = v.target.result;
      if (str) {
        importJson(str)
      }
    }
    reader.readAsText(file);
  } else if (file.name.endsWith(".zip")) {
    try {
      importLoading = true
      const JSZip = await loadJsLib('JSZip', `${Origin}/libs/jszip.min.js`);
      const zip = await JSZip.loadAsync(file);

      const dataFile = zip.file("data.json");
      if (!dataFile) {
        return Toast.error("缺少 data.json，导入失败");
      }

      const mp3Folder = zip.folder("mp3");
      if (mp3Folder) {
        const records: { id: string; file: Blob }[] = [];
        for (const filename in zip.files) {
          if (filename.startsWith("mp3/") && filename.endsWith(".mp3")) {
            const entry = zip.file(filename);
            if (!entry) continue;
            const blob = await entry.async("blob");
            const id = filename.replace(/^mp3\//, "").replace(/\.mp3$/, "");
            records.push({id, file: blob});
          }
        }
        await set(LOCAL_FILE_KEY, records);
      }

      const str = await dataFile.async("string");
      importJson(str, false)

      Toast.success("导入成功！");
    } catch (e) {
      Toast.error(e?.message || e || '导入失败')
    } finally {
      importLoading = false
    }
  } else {
    Toast.error("不支持的文件类型");
  }
}

let isNewHost = $ref(window.location.host === Host)

let showTransfer = $ref(false)

function transferOk() {
  setTimeout(() => {
    window.location.href = '/words'
  }, 1500)
}
</script>

<template>
  <BasePage>
    <div class="setting text-md card flex flex-col" style="height: calc(100vh - 3rem);">
      <div class="page-title text-align-center">设置</div>
      <div class="flex flex-1 overflow-hidden gap-4">
        <div class="left">
          <div class="tabs">
            <div class="tab" :class="tabIndex === 3 && 'active'" @click="tabIndex = 3">
              <IconFluentKeyboardLayoutFloat20Regular width="20"/>
              <span>快捷键设置</span>
            </div>
            <div class="tab" :class="tabIndex === 4 && 'active'" @click="tabIndex = 4">
              <IconFluentDatabasePerson20Regular width="20"/>
              <span>数据管理</span>
            </div>
            <div class="tab" :class="tabIndex === 5 && 'active'" @click="()=>{
            tabIndex = 5
            runtimeStore.isNew = false
            set(APP_VERSION.key,APP_VERSION.version)
          }">
              <IconFluentTextBulletListSquare20Regular width="20"/>
              <span>更新日志</span>
              <div class="red-point" v-if="runtimeStore.isNew"></div>
            </div>
            <div class="tab" :class="tabIndex === 6 && 'active'" @click="tabIndex = 6">
              <IconFluentPerson20Regular width="20"/>
              <span>关于</span>
            </div>
          </div>
        </div>
        <div class="col-line"></div>
        <div class="flex-1  overflow-y-auto overflow-x-hidden pr-4 content">

          <div class="body" v-if="tabIndex === 3">
            <div class="row">
              <label class="main-title">功能</label>
              <div class="wrapper">快捷键(点击可修改)</div>
            </div>
            <div class="scroll">
              <div class="row" v-for="item of Object.entries(settingStore.shortcutKeyMap)">
                <label class="item-title">{{ getShortcutKeyName(item[0]) }}</label>
                <div class="wrapper" @click="editShortcutKey = item[0]">
                  <div class="set-key" v-if="editShortcutKey === item[0]">
                    <input ref="shortcutInput" :value="item[1]?item[1]:'未设置快捷键'" readonly type="text"
                           @blur="handleInputBlur">
                    <span @click.stop="editShortcutKey = ''">按键盘进行设置，<span
                        class="text-red!">设置完成点击这里</span></span>
                  </div>
                  <div v-else>
                    <div v-if="item[1]">{{ item[1] }}</div>
                    <span v-else>未设置快捷键</span>
                  </div>
                </div>
              </div>
            </div>
            <div class="row">
              <label class="item-title"></label>
              <div class="wrapper">
                <BaseButton @click="resetShortcutKeyMap">恢复默认</BaseButton>
              </div>
            </div>
          </div>

          <div v-if="tabIndex === 4">
            <div>
              目前用户的所有数据
              <b class="text-red">仅保存在本地</b>。如果您需要在不同的设备、浏览器或者其他非官方部署上使用 {{ APP_NAME }}，
              您需要手动进行数据同步和保存。
            </div>
            <BaseButton :loading="exportLoading" class="mt-3" @click="exportData()">导出数据</BaseButton>

            <div class="line my-3"></div>

            <div>请注意，导入数据后将<b class="text-red"> 完全覆盖 </b>当前所有数据，请谨慎操作。
            </div>
            <div class="flex gap-space mt-3">
              <div class="import hvr-grow">
                <BaseButton :loading="importLoading">导入数据</BaseButton>
                <input type="file"
                       accept="application/json,.zip,application/zip"
                       @change="importData">
              </div>
            </div>

            <template v-if="isNewHost">
              <div class="line my-3"></div>
              <div>请注意，如果本地已有使用记录，请先备份当前数据，迁移数据后将<b class="text-red"> 完全覆盖 </b>当前所有数据，请谨慎操作。
              </div>
              <div class="flex gap-space mt-3">
                <BaseButton @click="showTransfer = true">迁移 2study.top 网站数据</BaseButton>
              </div>
            </template>
          </div>

          <!--          日志-->
          <div v-if="tabIndex === 5">
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/12/3</div>
                  <div>内容：单词、文章设置修改为弹框，更方便</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/12/3</div>
                  <div>内容：录入新概念（三、四）部分音频，优化文章相关功能</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/12/2</div>
                  <div>内容：完成新概念（一）音频，优化文章管理页面</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/30</div>
                  <div>内容：文章里的单词可点击播放</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/29</div>
                  <div>内容：修改 Slider 组件显示bug，新增 IE 浏览器检测提示</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/28</div>
                  <div>内容：新增引导框、 新增<a href="https://github.com/zyronon/TypeWords/pull/175" target="_blank">词典测试模式（由大佬
                    hebeihang 开发）</a></div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/25</div>
                  <div>内容：文章练习新增人名忽略功能（新概念一已全部适配），上传了新概念（一）1-18 音频</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/23</div>
                  <div>内容：优化练习完成结算界面，新增分享功能</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/22</div>
                  <div>内容：适配移动端</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/16</div>
                  <div>内容：辨认单词时，不认识单词可以直接输入，自动标识为错误单词，无需按2</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/15</div>
                  <div>内容：练习单词时，底部工具栏新增“跳到下一阶段”按钮</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/14</div>
                  <div>内容：新增文章练习时可跳过空格：如果在单词的最后一位上，不按空格直接输入下一个字母的话，自动跳下一个单词，
                    按空格也自动跳下一个单词
                  </div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/13</div>
                  <div>内容：新增文章练习时“输入时忽略符号/数字”选项</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/11/6</div>
                  <div>内容：新增随机复习功能</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/10/30</div>
                  <div>内容：集成PWA基础配置，支持用户以类App形式打开项目</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/10/26</div>
                  <div>内容：进一步完善单词练习，解决复习数量太多的问题</div>
                </div>
                <div class="text-base mt-1">
                  <ol>
                    <li>
                      <div class="title"><b>智能模式优化</b></div>
                      <div class="desc">练习时新增四种练习模式：学习、辨认、听写、默写。</div>
                    </li>
                    <li>
                      <div class="title"><b>学习模式</b></div>
                      <div class="desc">
                        <ul>
                          <li>仅在练习新词时出现。</li>
                          <li>采用「跟写 / 拼写」方式进行学习。</li>
                          <li>每 7 个单词会 <b>强制进行听写</b>，解决原来“一次练太多，听写时已忘记”的问题。</li>
                        </ul>
                      </div>
                    </li>
                    <li>
                      <div class="title"><b>辨认模式（新增）</b></div>
                      <div class="desc">
                        <ul>
                          <li>仅在复习已学单词时出现。</li>
                          <li>不再强制拼写，提供「我认识」与「不认识」选项。</li>
                          <li>选择「我认识」后，该单词在后续听写或默写中将不再出现，<b>显著减少复习数量</b>。</li>
                        </ul>
                      </div>
                    </li>
                    <li>
                      <div class="title"><b>听写模式</b></div>
                      <div class="desc">原有逻辑保持不变。</div>
                    </li>
                    <li>
                      <div class="title"><b>默写模式（新增）</b></div>
                      <div class="desc">
                        <ul>
                          <li>仅显示释义，不自动发音，不显示单词长度。</li>
                          <li>适合强化拼写记忆的场景。</li>
                        </ul>
                      </div>
                    </li>
                  </ol>
                  <b>说明：</b>
                  <div>本次更新重点解决了“复习单词数量过多、效率偏低”的问题。</div>
                  <div>通过引入「复习」与「默写」两种模式，使复习流程更加灵活、高效。</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/10/8</div>
                  <div>内容：文章支持自动播放下一篇</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/9/14</div>
                  <div>内容：完善文章编辑、导入、导出等功能</div>
                </div>
                <div class="text-base mt-1">
                  <div>1、文章的音频管理功能，目前已可添加音频、设置句子与音频的对应位置</div>
                  <div>2、文章可导入、导出</div>
                  <div>3、单词可导入、导出</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/8/10</div>
                  <div>内容：2.0版本发布，全新UI，全新逻辑，新增短语、例句、近义词等功能</div>
                </div>
              </div>
            </div>
            <div class="log-item">
              <div class="mb-2">
                <div>
                  <div>日期：2025/7/19</div>
                  <div>内容：1.0版本发布</div>
                </div>
              </div>
            </div>
          </div>

          <div v-if="tabIndex === 6" class="center flex-col">
            <h1>Type Words</h1>
            <p class="w-100 text-xl">
              感谢使用本项目！本项目是开源项目，如果觉得有帮助，请在 GitHub 点个 Star，您的支持是我持续改进的动力。
            </p>
            <p>
              GitHub地址：<a :href="GITHUB" target="_blank">{{ GITHUB }}</a>
            </p>
            <p>
              作者邮箱：<a :href="`mailto:${EMAIL}`">{{ EMAIL }}</a>
            </p>
            <div class="text-md color-gray mt-10">
              Build {{ gitLastCommitHash }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </BasePage>

  <MigrateDialog
      v-model="showTransfer"
      @ok="transferOk"
  />
</template>

<style scoped lang="scss">

.log-item {
  border-bottom: 1px solid var(--color-input-border);
  margin-bottom: 1rem;
}

.col-line {
  border-right: 2px solid gainsboro;
}

.setting {

  .left {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    align-items: center;

    .tabs {
      padding: .6rem 0;
      display: flex;
      flex-direction: column;
      gap: .6rem;

      .tab {
        @apply cursor-pointer flex items-center relative;
        padding: .6rem .9rem;
        border-radius: .5rem;
        width: 10rem;
        gap: .6rem;
        transition: all .5s;

        &:hover {
          background: var(--btn-primary);
          color: white;
        }

        &.active {
          background: var(--btn-primary);
          color: white;
        }
      }
    }
  }

  .content {
    .row {
      min-height: 2.6rem;
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: calc(var(--space) * 5);

      .wrapper {
        height: 2rem;
        flex: 1;
        display: flex;
        justify-content: flex-end;
        gap: var(--space);

        span {
          text-align: right;
          //width: 30rem;
          font-size: .7rem;
          color: gray;
        }

        .set-key {
          align-items: center;

          input {
            width: 9rem;
            box-sizing: border-box;
            margin-right: .6rem;
            height: 1.8rem;
            outline: none;
            font-size: 1rem;
            border: 1px solid gray;
            border-radius: .2rem;
            padding: 0 .3rem;
            background: var(--color-second);
            color: var(--color-font-1);
          }

        }
      }

      .main-title {
        font-size: 1.1rem;
        font-weight: bold;
      }

      .item-title {
        font-size: 1rem;
      }

      .sub-title {
        font-size: .9rem;
      }
    }

    .body {
      height: 100%;
      overflow: hidden;
      display: flex;
      flex-direction: column;
    }

    .scroll {
      flex: 1;
      padding-right: .6rem;
      overflow: auto;
    }

    .line {
      border-bottom: 1px solid #c4c3c3;
    }
  }
}

.import {
  display: inline-flex;
  position: relative;

  input {
    position: absolute;
    height: 100%;
    width: 100%;
    opacity: 0;
  }
}

// 移动端适配
@media (max-width: 768px) {
  .setting {
    flex-direction: column;

    .left {
      width: 100%;
      border-right: none;
      border-bottom: 2px solid gainsboro;

      .tabs {
        flex-direction: row;
        overflow-x: auto;
        padding: 0.5rem;
        gap: 0.3rem;

        .tab {
          white-space: nowrap;
          padding: 0.4rem 0.6rem;
          font-size: 0.9rem;

          span {
            display: none;
          }
        }
      }
    }

    .content {
      padding: 0 1rem;

      .row {
        flex-direction: column;
        align-items: flex-start;
        gap: 0.5rem;
        min-height: auto;
        padding: 0.5rem 0;

        .wrapper {
          width: 100%;
          justify-content: flex-start;

          .set-key {
            width: 100%;

            input {
              width: 100%;
              max-width: 200px;
            }
          }

          // 补充：选择器和输入框优化
          .base-select, .base-input {
            width: 100% !important;
            max-width: none;
          }

          // 单选按钮组优化
          .radio-group {
            flex-direction: column;
            gap: 0.5rem;

            .radio {
              min-height: 44px;
              width: 100%;
            }
          }

          // 滑块优化
          .slider {
            width: 100%;
          }
        }

        .main-title {
          font-size: 1rem;
        }

        .item-title {
          font-size: 0.9rem;
        }
      }

      .body {
        height: auto;
        max-height: 60vh;
      }
    }
  }
}
</style>
