<script lang="ts" setup>
import {onBeforeMount, reactive, ref} from 'vue'
import {del, get, patch, put} from "../api/http";
import {toggleDark} from "../composables";
import {BrowserOpenURL, ClipboardSetText, WindowSetDarkTheme, WindowSetLightTheme} from "../../wailsjs/runtime";
import {
  ExportConfig,
  GetFreePort,
  GetMacAcStatus,
  GetSecret,
  ImportConfig,
  IsAdmin,
  IsMac,
  IsNeedUpdate,
  OpenConfigDirectory,
  SetMacAc, SfQuit
} from "../../wailsjs/go/main/App";
import {ElLoading, ElMessage, ElMessageBox} from "element-plus";

const form = reactive({
  mode: "rule",
  tun: "off",
  mix_port: 10000,
  allow_lan: false,
  ipv6: false,

  is_dark: false,
  system_proxy: false,
  boot_start: false,
  clash_api: '',
  clash_secret: ''
})

const showTun = ref(false)

async function getConfig() {
  try {
    const temp = await get<any>("/configs")
    form.mode = temp.mode
    form.mix_port = temp["mixed-port"]
    form.allow_lan = temp["allow-lan"]
    form.ipv6 = temp.ipv6

    const TUN = temp["tun"]
    if (TUN["enable"]) {
      form.tun = TUN["stack"]
    } else {
      form.tun = "off"
    }
  } catch (error) {
    console.error(error);
  }
}

async function patchConfig(ogj: any) {
  try {
    await patch("/configs", ogj)
    if (ogj['mixed-port']) {
      localStorage.setItem("system_proxy_port", form.mix_port + "")
      if (form.system_proxy) {
        await put<any>(`/system/${form.mix_port}`)
      }
    }
  } catch (error) {
    console.error(error);
  }
}

async function setTun() {
  await patchConfig({tun: {enable: false}})
  if (form.tun != "off") {
    await patchConfig({tun: {enable: true, "stack": form.tun}})
  } else {
    setTimeout(() => del("/connections"), 3000)
  }

  localStorage.setItem("tun", form.tun)
}

function setIpv6() {
  patchConfig({ipv6: form.ipv6})
  localStorage.setItem("ipv6", form.ipv6 ? "0" : "1")
}

async function setSystemProxy() {
  if (form.system_proxy) {
    await put<any>(`/system/${form.mix_port}`)
  } else {
    await del<any>(`/system`)
  }
  localStorage.setItem("system_proxy_port", form.mix_port + "")
  localStorage.setItem("system_proxy", form.system_proxy ? "0" : "1")
}

function setDark() {
  toggleDark()
  if (form.is_dark) {
    WindowSetDarkTheme()
  } else {
    WindowSetLightTheme()
  }
}

async function openConfig() {
  await OpenConfigDirectory()
}

async function importConfig() {
  const loading = ElLoading.service({
    lock: true,
    text: '导入中Importing...',
    background: 'rgba(0, 0, 0, 0.7)',
  })
  const ok = await ImportConfig()
  loading.close()

  switch (ok) {
    case "false":
      break;
    case "true":
      ElMessage({
        showClose: true,
        message: "导入成功 Import Success",
        type: 'success',
      })
      break;
    default:
      ElMessage({
        showClose: true,
        message: "导入失败 Import failed : " + ok,
        type: 'error',
      })
      break;
  }
}

async function exportConfig() {

  const loading = ElLoading.service({
    lock: true,
    text: '导出中Exporting...',
    background: 'rgba(0, 0, 0, 0.7)',
  })

  const ok = await ExportConfig()
  loading.close()

  switch (ok) {
    case "false":
      break;
    case "true":
      ElMessage({
        showClose: true,
        message: "导出成功 Export Success",
        type: 'success',
      })
      break;
    default:
      ElMessage({
        showClose: true,
        message: "导出失败 Export failed : " + ok,
        type: 'error',
      })
      break;
  }
}

function copyApi() {
  ClipboardSetText(form.clash_api)
  ElMessage({
    showClose: true,
    message: "复制成功 Copy Success",
    type: 'success',
  })
}

function copySecret() {
  ClipboardSetText(form.clash_secret)
  ElMessage({
    showClose: true,
    message: "复制成功 Copy Success",
    type: 'success',
  })
}

async function checkUpdate() {
  const loading = ElLoading.service({
    lock: true,
    text: '检查中Checking...',
    background: 'rgba(0, 0, 0, 0.7)',
  })

  const isNeedUpdate = await IsNeedUpdate()
  loading.close()

  if (isNeedUpdate === "true") {
    ElMessageBox.confirm(
        '发现新版本，前往下载 Found a new version, Continue?',
        '更新提示 Warning',
        {
          confirmButtonText: '确定 OK',
          cancelButtonText: '取消 Cancel',
          type: 'warning',
        }
    )
        .then(() => {
          BrowserOpenURL('https://github.com/snakem982/Pandora-Box/releases/latest')
        })
  } else {
    ElMessage({
      message: "已经是最新版本 No updates available",
      type: 'success',
    })
  }
}

async function quit() {
  await SfQuit()
}


const pwdVisible = ref(false)
const pwd = ref("")
const isMac = ref(false)
const acStatusVal = ref("点击授权 Click Authorize")
const acStatusX = ref("1")

function authorize() {
  if (isMac.value) {
    if (acStatusX.value == "3") {
      ElMessage({
        showClose: true,
        message: "授权成功,请关闭软件后重新打开,即可使用Tun。 Authorization is successful. Please close the software and reopen it to use Tun",
        type: 'success',
        duration: 10000,
      })
      return
    }
    pwdVisible.value = true
  } else {
    ElMessageBox.alert('请关闭软件，然后右键以管理员身份运行<br>Please close the software and run it as an administrator', '提示 Tip', {
      confirmButtonText: '确认 Confirm',
      center: true,
      type: 'warning',
      dangerouslyUseHTMLString: true,
    })
  }
}

async function authorizeConfirm() {
  const temp = pwd.value.trim()
  if (temp.length == 0) {
    ElMessage({
      showClose: true,
      message: "密码为空 Password is empty",
      type: 'error',
    })
    return
  }
  const ac = await SetMacAc(temp)
  if (ac == "1") {
    ElMessage({
      showClose: true,
      message: "密码错误 Password error",
      type: 'error',
      duration: 5000,
    })
    return
  } else if (ac == "2") {
    ElMessage({
      showClose: true,
      message: "授权失败，请稍后再试 Authorization failed, please try again later",
      type: 'error',
      duration: 5000,
    })
    return
  } else {
    acStatusX.value = "3"
    ElMessage({
      showClose: true,
      message: "授权成功,请关闭软件后重新打开,即可使用Tun。 Authorization is successful. Please close the software and reopen it to use Tun",
      type: 'success',
      duration: 10000,
    })
  }

  pwdVisible.value = false
}

onBeforeMount(async () => {
  const baseUrl = await GetFreePort()
  form.clash_api = 'http://' + baseUrl

  const secret = await GetSecret()
  form.clash_secret = secret

  form.is_dark = localStorage.getItem("vueuse-color-scheme") == "dark"

  const mac = await IsMac()
  isMac.value = mac == "true"

  await getConfig()

  const isAdmin = await IsAdmin()
  if (isAdmin === "true") {
    showTun.value = true
    const tun = localStorage.getItem("tun")
    if (tun && form.tun != tun) {
      form.tun = tun
      await setTun()
    }
  } else {
    showTun.value = false
    if (isMac.value) {
      const acStatus = await GetMacAcStatus()
      acStatusX.value = acStatus
      if (acStatus == "2") {
        acStatusVal.value = "点击重新授权 Click Reauthorization"
      }
    }
  }

  const system_proxy_port = localStorage.getItem("system_proxy_port");
  if (system_proxy_port) {
    const number = Number(system_proxy_port);
    if (form.mix_port != number) {
      patch("/configs", {'mixed-port': number})
    }
    form.mix_port = number
  }

  const system_proxy = localStorage.getItem("system_proxy") == "0"
  if (form.system_proxy != system_proxy && system_proxy && system_proxy_port) {
    put<any>("/system/" + system_proxy_port)
  }
  form.system_proxy = system_proxy

  const ipv6 = localStorage.getItem("ipv6") == "0"
  if (form.ipv6 != ipv6) {
    form.ipv6 = ipv6
    setIpv6()
  }
})


</script>

<template>
  <el-card class="box-card">
    <template #header>
      <div class="card-header">
        <span>代理 Proxy</span>
      </div>
    </template>
    <div class="text item">
      <el-form label-position="left" label-width="230px">
        <el-form-item label="混合代理端口 mix-port">
          <el-input-number
              v-model="form.mix_port"
              :min="1"
              :max="65000"
              :controls="false"
              @blur="patchConfig({'mixed-port':form.mix_port})"
          />
        </el-form-item>
        <el-form-item label="开启TUN" v-if="showTun">
          <el-select v-model="form.tun" @change="setTun" style="width: 150px">
            <el-option label="关闭 Off" value="off"/>
            <el-option label="System" value="System"/>
            <el-option label="gVisor" value="gVisor"/>
            <el-option label="Mixed" value="Mixed"/>
          </el-select>
        </el-form-item>
        <el-form-item label="开启TUN" v-else>
          <el-button type="primary" size="small" @click="authorize" round>{{ acStatusVal }}</el-button>
        </el-form-item>
        <el-form-item label="允许局域网连接 allow-lan">
          <el-switch v-model="form.allow_lan" @change="patchConfig({'allow-lan':form.allow_lan,'bind-address':'*'})"/>
        </el-form-item>
        <el-form-item label="开启ipv6">
          <el-switch v-model="form.ipv6" @change="setIpv6"/>
        </el-form-item>
      </el-form>
    </div>
  </el-card>
  <br>
  <el-card class="box-card">
    <template #header>
      <div class="card-header">
        <span>系统 System</span>
      </div>
    </template>
    <div class="text item">
      <el-form label-position="left" label-width="230px">
        <el-form-item label="暗黑模式 dark mode">
          <el-switch v-model="form.is_dark" @click="setDark"/>
        </el-form-item>
        <el-form-item label="设置为系统代理 system proxy">
          <el-switch v-model="form.system_proxy" @click="setSystemProxy"/>
        </el-form-item>
        <el-form-item label="配置目录 configs directory">
          <el-button type="primary" size="small" @click="openConfig" round>打开 open</el-button>
          <el-button type="danger" size="small" @click="exportConfig" round>导出 export</el-button>
          <el-button type="success" size="small" @click="importConfig" round>导入 import</el-button>
        </el-form-item>
        <el-form-item label="控制面板 clash api">
          <el-text>{{ form.clash_api }}</el-text>&emsp;&emsp;
          <el-button type="primary" size="small" @click="copyApi" round>复制 copy</el-button>
        </el-form-item>
        <el-form-item label="控制密钥 clash secret">
          <el-text>{{ form.clash_secret }}</el-text>&emsp;&emsp;
          <el-button type="primary" size="small" @click="copySecret" round>复制 copy</el-button>
        </el-form-item>
        <el-form-item label="软件操作 software option">
          <el-button type="primary" size="small" @click="checkUpdate" round>检查更新 check update</el-button>
          <el-button type="danger" size="small" @click="quit" round>退出软件 quit software</el-button>
        </el-form-item>
      </el-form>
    </div>
  </el-card>


  <el-dialog v-model="pwdVisible" title="授权 Authorize" width="400" center>
    <el-text>请在下面方框中输入管理员密码</el-text>
    <br>
    <el-text>Please input the administrator password</el-text>
    <br><br>
    <el-input v-model="pwd"
              autocomplete="off"
              type="password"
              show-password/>
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="pwdVisible = false">取消 Cancel</el-button>
        <el-button type="primary" @click="authorizeConfirm">
          确认 Confirm
        </el-button>
      </div>
    </template>
  </el-dialog>
</template>

<style>
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.text {
  font-size: 14px;
}

.item {
  margin-bottom: 18px;
}

.box-card {
  width: 98%;
  margin: auto;
}
</style>
