<template>
  <div id="live">
    <danmaku-list
      ref="giftPinList"
      v-bind="props"
      :gift-show-face="showFace"
      :is-gift-list="true"
      v-if="props.giftPin"
    />
    <danmaku-list ref="danmakuList" v-bind="props" />
  </div>
</template>

<script>
import { onBeforeUnmount, ref, onMounted, computed } from 'vue';
import { KeepLiveWS } from 'bilibili-live-ws';
import { propsType } from '@/utils/props';

import DanmakuList from '@/components/DanmakuList';

export default {
  components: { DanmakuList },
  props: {
    ...propsType,
    anchor: String,
    liveWsOptions: Object,
  },
  setup(props) {
    const giftPinList = ref(null);
    const danmakuList = ref(null);

    const giftCombMap = new Map();
    const showFace = computed(() => props.face !== 'false');

    const blockUIDs = computed(() => new Set(props.blockUID.split(/,|\|/).map(uid => uid.trim())));
    const isBlockedUID = uid => blockUIDs.value.has(String(uid));

    let failedTimestamps = [];

    const addInfoDanmaku = message => {
      danmakuList.value.addDanmaku({
        type: 'info',
        message,
        stay: props.stay || 5000,
      });
    };
    const addDanmaku = danmaku => {
      if (props.limit) danmakuList.value.addSpeedLimitDanmaku(danmaku);
      else danmakuList.value.addDanmaku(danmaku);
    };

    onMounted(() => {
      console.log('正在连接直播弹幕服务器', props.room);
      const live = new KeepLiveWS(props.room, props.liveWsOptions || { protover: 3, uid: 0 });
      live.interval = 1000;
      onBeforeUnmount(() => live.close());
      live.on('open', () => {
        if (live.closed) return;
        console.log('已连接直播弹幕服务器');
        addInfoDanmaku('已连接直播弹幕服务器');
      });
      live.on('live', () => {
        if (live.closed) return;
        console.log('已连接直播间', props.room);
        addInfoDanmaku(`已连接直播间 ${props.room}`);
      });
      live.on('close', () => {
        if (live.closed) return;
        console.log('连接已断开');
        addInfoDanmaku('连接已断开');
        const now = Date.now();
        failedTimestamps = failedTimestamps.filter(time => now - time < 10000);
        failedTimestamps.push(now);
        if (failedTimestamps.length >= 3) {
          console.log('连接失败过于频繁，停止重连');
          addInfoDanmaku('连接失败过于频繁，停止重连');
          live.close();
        }
      });

      // 礼物
      const giftList = props.giftPin ? giftPinList : danmakuList;
      live.on('LIVE_OPEN_PLATFORM_SEND_GIFT', ({ data }) => {
        handleSendGift({
          open_id: data.open_id,
          uname: data.uname,
          giftName: data.gift_name,
          num: data.gift_num,
          face: data.uface,
        });
      });
      const handleSendGift = ({ open_id, uname, gift_name, gift_num, face }) => {
        if (isBlockedUID(open_id)) {
          console.log(`屏蔽了来自[${uname}]的礼物：${gift_name}*${gift_num}`);
          return;
        }
        const giftData = {
          type: 'gift',
          showFace: showFace.value,
          open_id,
          uname,
          giftName: gift_name,
          num: gift_num,
          face,
        };
        if (props.giftComb) {
          const key = `${open_id}-${gift_name}`;
          const existComb = giftCombMap.get(key);
          if (existComb) {
            giftCombMap.set(key, {
              ...existComb,
              num: existComb.num + gift_num,
            });
          } else {
            giftCombMap.set(key, giftData);
            setTimeout(() => {
              giftList.value.addDanmaku(giftCombMap.get(key));
              giftCombMap.delete(key);
            }, props.giftComb);
          }
        } else {
          giftList.value.addDanmaku(giftData);
        }
      };

      // 弹幕
      live.on('LIVE_OPEN_PLATFORM_DM', ({ data }) => {
        console.log(data);
        handleDanmaku(data);
      });
      const handleDanmaku = ({ open_id, uname, message, uface, is_admin }) => {
        if (isBlockedUID(open_id)) {
          console.log(`屏蔽了来自[${uname}]的弹幕：${message}`);
          return;
        }
        const danmaku = {
          type: 'message',
          showFace: showFace.value,
          open_id: open_id,
          uname,
          message,
          isAnchor: open_id === props.anchor,
          isAdmin: !!is_admin,
          face: uface,
        };
        if (props.delay > 0) setTimeout(() => addDanmaku(danmaku), props.delay * 1000);
        else addDanmaku(danmaku);
      };

      // SC
      live.on('LIVE_OPEN_PLATFORM_SUPER_CHAT', ({ data: { uid, uname, message, uface } }) => {
        handleSuperChat({ uid, uname, message, face: uface });
      });
      const handleSuperChat = ({ uid, uname, message, face }) => {
        giftList.value.addDanmaku({
          type: 'sc',
          showFace: showFace.value,
          uid,
          uname,
          message,
          face,
        });
      };

      // 上舰
      const guardLevelMap = { 1: '总督', 2: '提督', 3: '舰长' };
      live.on('LIVE_OPEN_PLATFORM_GUARD', ({ data }) => {
        handleGuard(data);
      });
      const handleGuard = ({ user_info: { open_id, uname, uface }, guard_level, guard_num, guard_unit }) => {
        const guardName = guardLevelMap[guard_level];
        giftList.value.addDanmaku({
          type: 'gift',
          showFace: showFace.value,
          open_id,
          uname,
          giftName: guard_unit ? `${guardName} * ${guard_num}${guard_unit}` : guardName,
          num: guard_unit ? 0 : guard_num,
          face: uface,
        });
      };
    });

    return { props, showFace, giftPinList, danmakuList };
  },
};
</script>

<style lang="scss">
#live {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  background-color: transparent;
  display: flex;
  flex-direction: column;
}
</style>
