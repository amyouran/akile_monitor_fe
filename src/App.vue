<script setup>
import {computed, onMounted, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import config from '@/config'

const area = ref([])
const selectArea = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }
  return data.value.filter(item => item.Host.Name.slice(0, 2) === selectArea.value)
})

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetInSpeed
    bandwidth_down += item.State.NetOutSpeed
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down
  }
})

let socket = null

let nowtime = (Math.floor(Date.now() / 1000))

const initScoket = () => {
  socket = new WebSocket(config.socket);  // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function(event) {
    try {
      const message = event.data;
      const res = JSON.parse(message.replace('data: ', ''))
      area.value = Array.from(new Set(res.map(item => item.Host.Name.slice(0, 2))))
      data.value = res.map((host) => {
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length == 60) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetInSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetOutSpeed])

        return {
          ...host,
          status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
        }
      })

      sendPing()
    } catch (error) {
      console.error('解析 WebSocket 消息时出错:', error);
    }
  };

  socket.onopen = function () {
    sendPing()
  }
}

const sendPing = () => {
  nowtime = (Math.floor(Date.now() / 1000))
  socket.send('ping')
}

onMounted(() => {
  initScoket()
})

// 格式化字节单位
const formatBytes = (bytes) => {
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  let i = 0;
  while (bytes >= 1024 && i < units.length - 1) {
    bytes /= 1024;
    i++;
  }
  return bytes.toFixed(2) + ' ' + units[i];
}

// 格式化字节单位
const formatMem = (bytes) => {
  const units = ['B', 'KB', 'MB', 'GB', 'TB'];
  let i = 0;
  while (bytes >= 1024 && i < units.length - 1) {
    bytes /= 1024;
    i++;
  }
  return bytes.toFixed(2) + ' ' + units[i];
}

// 格式化系统时间为小时:分钟:秒
const formatTime = (seconds) => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const remainingSeconds = seconds % 60;
  return `${hours}小时 ${minutes}分钟 ${remainingSeconds}秒`;
}

// 格式化时间戳为可读的日期格式
const formatTimeStamp = (timestamp) => {
  const date = new Date(timestamp * 1000); // 转换为毫秒
  const year = date.getFullYear();
  const month = ('0' + (date.getMonth() + 1)).slice(-2);
  const day = ('0' + date.getDate()).slice(-2);
  const hours = ('0' + date.getHours()).slice(-2);
  const minutes = ('0' + date.getMinutes()).slice(-2);
  const seconds = ('0' + date.getSeconds()).slice(-2);
  return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
}

const formatUptime = (seconds) => {
  const days = Math.floor(seconds / (24 * 3600));
  seconds %= 24 * 3600;
  const hours = Math.floor(seconds / 3600);
  seconds %= 3600;
  const minutes = Math.floor(seconds / 60);
  const secs = seconds % 60;

  return `${days}d ${hours}h ${minutes}m ${secs}s`;
}

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'error';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

</script>

<template>
  <div class="max-container">
    <a class="logo" href="#">
      <svg class="arco-icon" viewBox="0 0 48 48" fill="currentColor">
        <path data-v-2ee6cb6b="" fill-rule="evenodd" clip-rule="evenodd" d="M42.919 11.923L25 1.577a2 2 0 00-2 0L5.081 11.923a2 2 0 00-1 1.732v20.69a2 2 0 001 1.732L23 46.423a2 2 0 002 0l17.919-10.346a2 2 0 001-1.732v-20.69a2 2 0 00-1-1.732zM30.556 9.525L38.5 14 24 23l-13.808-8.668L17.5 10l6.5 4 6.556-4.475zM22 40.441V26.286L8 17.358v7.928l8 5.464v6.227l6 3.464zm10-3.464l-6 3.464V26.286l14-8.928v8.928l-8 5.464v5.227z" fill="currentColor"></path>
      </svg>
      <span>Akile Monitor</span>
      <small style="font-weight: 400;opacity: .8"> ｜ 全球节点监控</small>
    </a>
    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        全部地区
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>
    <div class="hero">
      <a-row :gutter="20">
        <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
          <div class="hero-card">
            <div class="title">服务器总数</div>
            <div class="value">
              <div class="status" style="background: #005fe7;"></div>
              <span class="num">{{stats.total}} 台</span>
            </div>
          </div>
        </a-col>
        <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
          <div class="hero-card">
            <div class="title">在线服务器</div>
            <div class="value">
              <div class="status" style="background: #1fb416;"></div>
              <span class="num">{{stats.online}} 台</span>
            </div>
          </div>
        </a-col>
        <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
          <div class="hero-card">
            <div class="title">离线服务器</div>
            <div class="value">
              <div class="status" style="background: #b41616;"></div>
              <span class="num">{{stats.offline}} 台</span>
            </div>
          </div>
        </a-col>
        <a-col :span="6" :xs="24" :sm="24" :md="6" :lg="6" :sl="6">
          <div class="hero-card">
            <div class="title">实时带宽</div>
            <div class="value">
              <icon-arrow-up style="font-size: 20px;" />
              <span style="font-size: 16px;"> {{formatBytes(stats.bandwidth_up)}}/s</span>

              <icon-arrow-down style="font-size: 20px;" />
              <span style="font-size: 16px;">{{formatBytes(stats.bandwidth_down)}}/s</span>
            </div>
          </div>
        </a-col>
      </a-row>
    </div>
    <div class="monitor-card">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in host" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status  ? '在线' : '离线'}}</span>
            <span style="margin-left: 6px;">{{formatUptime(item.State.Uptime)}}</span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">系统</div>
          <div class="monitor-item-value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">内存使用情况</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">网络速度（IN|OUT）</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">负载平均值(1|5|15)</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <div class="uptime">
          <div class="monitor-item-title">上报时间</div>
          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>
        </div>
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">主机名</div>
                  <div class="value">{{item.Host.Name}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">地区</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Name.slice(0, 2).toUpperCase()}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">系统</div>
                  <div class="value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">架构</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">虚拟化</div>
                  <div class="value">{{item.Host.Virtualization || '-'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU占用</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">内存使用情况</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatMem(item.State.MemUsed)}} / {{formatMem(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">网络速度（IN|OUT）</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">负载平均值(1|5|15)</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">流量使用↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetInTransfer)}} | {{formatBytes(item.State.NetOutTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">开机时间</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">上报时间</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <CPU ref="cpuRef" style="margin-bottom: 20px;" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" style="margin-bottom: 20px;" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
      </div>
    </div>
    <div class="footer">Copyright © 2023-{{new Date().getFullYear()}} Akile LTD.</div>
  </div>
</template>

<style lang="scss" scoped>
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    transition: .14s all ease-in-out;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.hero {
  margin: 30px 10px 0px 10px;

  .hero-card {
    margin-bottom: 20px;
    padding: 12px 24px;
    border: 1px solid #e5e5e5;
    border-radius: 6px;
    background: #ffffff;
    min-height: 62px;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);

    .title {
      margin-top: 6px;
      font-size: 16px;
      font-weight: 500;
      margin-bottom: 14px;
    }

    .value {
      display: flex;
      align-items: center;
      .status {
        margin-right: 6px;
        width: 8px;
        height: 8px;
        border-radius: 10px;
        background: #333333;
      }

      .num {
        font-size: 20px;
        font-weight: 600;
      }
    }
  }
}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;
    transition: .14s all ease-in-out;

    &.is-active {
      background: #e7e7e730;
      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e7e7e730;
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;
        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: .6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .uptime {
      display: inline-block;
      vertical-align: middle;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 20%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 80%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  margin-left: 13px;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .arco-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22,93,255)!important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  margin-top: 15px;
  margin-bottom: 30px;
  text-align: center;
  color: #565656;
}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {

    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}
</style>
