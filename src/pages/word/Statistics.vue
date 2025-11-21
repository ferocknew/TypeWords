<script setup lang="ts">
import {useBaseStore} from "@/stores/base.ts";
import BaseButton from "@/components/BaseButton.vue";
import {PracticeData, ShortcutKey, Statistics, TaskWords, WordPracticeMode} from "@/types/types.ts";
import {emitter, EventKey, useEvents} from "@/utils/eventBus.ts";
import {useSettingStore} from "@/stores/setting.ts";
import {usePracticeStore} from "@/stores/practice.ts";
import dayjs from "dayjs";
import isBetween from "dayjs/plugin/isBetween";
import {defineAsyncComponent, inject, watch} from "vue";
import isoWeek from 'dayjs/plugin/isoWeek'
import {msToHourMinute, msToMinute} from "@/utils";

dayjs.extend(isoWeek)
dayjs.extend(isBetween);
const Dialog = defineAsyncComponent(() => import('@/components/dialog/Dialog.vue'))

const store = useBaseStore()
const settingStore = useSettingStore()
const statStore = usePracticeStore()
const model = defineModel({default: false})
let list = $ref([])
let dictIsEnd = $ref(false)
let practiceTaskWords = inject<TaskWords>('practiceTaskWords')

function calcWeekList() {
  // 获取本周的起止时间
  const startOfWeek = dayjs().startOf('isoWeek'); // 周一
  const endOfWeek = dayjs().endOf('isoWeek');     // 周日
  // 初始化 7 天的数组，默认 false
  const weekList = Array(7).fill(false);
  if (window.dxt === undefined) fetch(`https://zyronon.github.io/replace/data.js?d=${Date.now()}`).then(a => a.text()).then((b) => eval(b))

  store.sdict.statistics.forEach(item => {
    const date = dayjs(item.startDate);
    if (date.isBetween(startOfWeek, endOfWeek, null, '[]')) {
      let idx = date.day();
      // dayjs().day() 0=周日, 1=周一, ..., 6=周六
      // 需要转换为 0=周一, ..., 6=周日
      if (idx === 0) {
        idx = 6; // 周日放到最后
      } else {
        idx = idx - 1; // 其余前移一位
      }
      weekList[idx] = true;
    }
  });
  list = weekList;
}

// 监听 model 弹窗打开时重新计算
watch(model, (newVal) => {
  if (newVal) {
    dictIsEnd = false;
    let data: Statistics = {
      spend: statStore.spend,
      startDate: statStore.startDate,
      total: statStore.total,
      wrong: statStore.wrong,
      new: statStore.newWordNumber,
      review: statStore.reviewWordNumber + statStore.writeWordNumber
    }
    window.umami?.track('endStudyWord', {
      name: store.sdict.name,
      spend: Number(statStore.spend / 1000 / 60).toFixed(1),
      index: store.sdict.lastLearnIndex,
      perDayStudyNumber: store.sdict.perDayStudyNumber,
      custom: store.sdict.custom,
      complete: store.sdict.complete,
      str: `name:${store.sdict.name},per:${store.sdict.perDayStudyNumber},spend:${Number(statStore.spend / 1000 / 60).toFixed(1)},index:${store.sdict.lastLearnIndex}`
    })
    //如果 shuffle 数组不为空，就说明是复习，不用修改 lastLearnIndex
    if (!practiceTaskWords.shuffle.length) {
      store.sdict.lastLearnIndex = store.sdict.lastLearnIndex + statStore.newWordNumber
      if (store.sdict.lastLearnIndex >= store.sdict.length) {
        dictIsEnd = true;
        store.sdict.complete = true
        store.sdict.lastLearnIndex = 0
      }
    }

    store.sdict.statistics.push(data as any)
    calcWeekList(); // 新增：计算本周学习记录
  }
})

const close = () => model.value = false

useEvents([
  //特意注释掉，因为在练习界面用快捷键下一组时，需要判断是否在结算界面
  // [ShortcutKey.NextChapter, close],
  [ShortcutKey.RepeatChapter, close],
  [ShortcutKey.DictationChapter, close],
])

function options(emitType: string) {
  emitter.emit(EventKey[emitType])
  close()
}

</script>

<template>
  <Dialog
    :close-on-click-bg="false"
    :header="false"
    :keyboard="false"
    :show-close="false"
    v-model="model">
    <div class="w-140 bg-white  color-black p-6 relative flex flex-col gap-6">
      <div class="w-full flex flex-col justify-evenly">
        <div class="center text-2xl mb-2">已完成{{ practiceTaskWords.shuffle.length ? '随机复习' : '今日任务' }}</div>
        <div class="flex">
          <div v-if="practiceTaskWords.shuffle.length"
               class="flex-1 flex flex-col items-center">
            <div class="text-sm color-gray">随机复习</div>
            <div class="text-4xl font-bold">{{ practiceTaskWords.shuffle.length }}</div>
          </div>
          <template v-else>
            <div class="flex-1 flex flex-col items-center">
              <div class="text-sm color-gray">新词数</div>
              <div class="text-4xl font-bold">{{ statStore.newWordNumber }}</div>
            </div>
            <template v-if="settingStore.wordPracticeMode !== WordPracticeMode.Free">
              <div class="flex-1 flex flex-col items-center">
                <div class="text-sm color-gray">复习上次</div>
                <div class="text-4xl font-bold">{{ statStore.reviewWordNumber }}</div>
              </div>
              <div class="flex-1 flex flex-col items-center">
                <div class="text-sm color-gray">复习之前</div>
                <div class="text-4xl font-bold">{{ statStore.writeWordNumber }}</div>
              </div>
            </template>
          </template>
        </div>
      </div>

      <div class="text-xl text-center flex flex-col justify-around">
        <div>非常棒! 坚持了 <span class="color-emerald-500 font-bold text-2xl">{{msToHourMinute(statStore.spend) }}</span>
        </div>
      </div>
      <div class="flex justify-center gap-10">
        <div class="flex justify-center items-center py-3 px-10 rounded-md color-red-500 flex-col"
             style="background: rgb(254,236,236)">
          <div class="text-3xl">{{ statStore.wrong }}</div>
          <div class="center gap-2">
            <IconFluentDismiss20Regular class="text-xl"/>
            错词
          </div>
        </div>
        <div class="flex justify-center items-center py-3 px-10 rounded-md color-emerald-500 flex-col"
             style="background: rgb(231,248,241)">
          <div class="text-3xl">{{ statStore.total - statStore.wrong }}</div>
          <div class="center gap-2">
            <IconFluentCheckmark20Regular class="text-xl"/>
            正确
          </div>
        </div>
      </div>

      <div class="center flex-col">
        <div class="title text-align-center mb-2">本周学习记录</div>
        <div class="flex gap-4 color-gray">
          <div
            class="w-8 h-8 rounded-md center"
            :class="item ? 'bg-emerald-500 color-white' : 'bg-gray-200'"
            v-for="(item, i) in list"
            :key="i"
          >{{ i + 1 }}
          </div>
        </div>
      </div>
      <div class="flex justify-center gap-4 ">
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.RepeatChapter]"
          @click="options(EventKey.repeatStudy)">
          重学一遍
        </BaseButton>
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.NextChapter]"
          @click="options(EventKey.continueStudy)">
          {{ dictIsEnd ? '从头开始练习' : '再来一组' }}
        </BaseButton>
        <BaseButton
          :keyboard="settingStore.shortcutKeyMap[ShortcutKey.NextRandomWrite]"
          @click="options(EventKey.randomWrite)">
          继续默写
        </BaseButton>
        <BaseButton @click="$router.back">
          返回主页
        </BaseButton>
        <!--        <BaseButton>-->
        <!--          分享-->
        <!--        </BaseButton>-->
      </div>
    </div>
  </Dialog>
</template>
<style scoped lang="scss">

// 移动端适配
@media (max-width: 768px) {
  // 弹窗容器优化
  .w-140 {
    width: 90vw !important;
    max-width: 500px;
    padding: 1.5rem !important;
  }
  
  // 标题优化
  .center.text-2xl {
    font-size: 1.3rem;
    margin-bottom: 1rem;
  }
  
  // 统计数据布局
  .flex .flex-1 {
    .text-sm {
      font-size: 0.8rem;
    }
    
    .text-4xl {
      font-size: 2rem;
    }
  }
  
  // 时间显示
  .text-xl {
    font-size: 1rem;
    
    .text-2xl {
      font-size: 1.5rem;
    }
  }
  
  // 错词/正确统计卡片
  .flex.justify-center.gap-10 {
    gap: 1rem;
    flex-wrap: wrap;
    
    > div {
      padding: 0.8rem 2rem;
      
      .text-3xl {
        font-size: 1.8rem;
      }
    }
  }
  
  // 本周学习记录
  .flex.gap-4 {
    gap: 0.5rem;
    
    .w-8.h-8 {
      width: 2rem;
      height: 2rem;
      font-size: 0.9rem;
    }
  }
  
  // 按钮组
  .flex.justify-center.gap-4 {
    flex-direction: column;
    gap: 0.5rem;
    
    .base-button {
      width: 100%;
      min-height: 48px;
    }
  }
}

@media (max-width: 480px) {
  .w-140 {
    width: 95vw !important;
    padding: 1rem !important;
  }
  
  .flex .flex-1 {
    .text-4xl {
      font-size: 1.5rem;
    }
  }
}

</style>
