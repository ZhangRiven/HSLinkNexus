<script setup lang="ts">

import {onMounted, reactive, ref, computed, watch} from "vue";
import {hslink_write_wait_rsp} from "../backend/hslink_backend.ts";
import {storeToRefs}from "pinia";
import {useDeviceStore}from "../stores/deviceStore.ts";
import {useI18n} from 'vue-i18n';

const {t} = useI18n();

const deviceStore = useDeviceStore()
const {connected} = storeToRefs(deviceStore);
const {sn, nickname, model, hw_ver, sw_ver, bl_ver} = storeToRefs(deviceStore)
const {
  speed_boost_enable, swd_simulate_mode, jtag_simulate_mode,
  power_power_on, power_port_on, power_vref_voltage,
  reset_mode, led_enable, led_brightness, jtag_single_bit_mode, jtag_20pin_compatible
} = storeToRefs(deviceStore)

const show_alert = ref(false)
const alert_msg = ref("")
const alert_type = ref("") // "success" 或 "error"

// 参考电压设置部分
const isExternalVref = computed({
  get: () => power_vref_voltage.value === 0,
  set: (value) => {
    if (value) {
      power_vref_voltage.value = 0; // 设置为外部输入
    } else {
      power_vref_voltage.value = 3.3; // 默认为3.3V
    }
  }
});

// 电压设置模式: "preset"(预设值) 或 "custom"(自定义)
const voltageMode = ref("preset");

// 预设电压值
const presetVoltages = [1.8, 3.3, 3.6, 5.0];

// 自定义电压值
const customVoltage = ref("");

// 电压输入错误信息
const voltageErrorMsg = ref("");

// 检查是否可以保存设置 - 使用 voltageErrorMsg 判断
const canSave = computed(() => {
  return connected.value && !voltageErrorMsg.value; // 如果没有错误信息，则认为有效
});

// 当选择预设值时更新电压
const selectPresetVoltage = (voltage: number) => {
  power_vref_voltage.value = voltage;
  voltageMode.value = "preset";
  customVoltage.value = voltage.toString();
  voltageErrorMsg.value = ""; // 清除错误信息
};

// 当自定义输入改变时更新电压并设置错误信息
const updateCustomVoltage = () => {
  console.log("更新电压值:", customVoltage.value);
  const valueStr = customVoltage.value.toString().trim();
  if (valueStr === "") {
    voltageErrorMsg.value = t('device_setting.voltage_errors.empty');
    return; // Exit early if empty
  }

  const value = Number(valueStr);

  if (isNaN(value)) {
    voltageErrorMsg.value = t('device_setting.voltage_errors.invalid');
    console.log("无效的数字:", valueStr);
  } else if (value < 1.8) {
    voltageErrorMsg.value = t('device_setting.voltage_errors.too_low');
    console.log("电压值太低:", value);
  } else if (value > 5.0) {
    voltageErrorMsg.value = t('device_setting.voltage_errors.too_high');
    console.log("电压值太高:", value);
  } else {
    // Valid input
    console.log("有效的输入值:", value);
    voltageErrorMsg.value = ""; // Clear error message
    const roundedValue = Math.round(value * 10) / 10; // Round to 1 decimal place
    console.log("更新电压值为:", roundedValue);
    power_vref_voltage.value = roundedValue; // Update store value
    voltageMode.value = "custom";
  }
};

// 切换到自定义电压输入模式
const switchToCustomMode = () => {
  voltageMode.value = "custom";
  // Trigger validation when switching to custom mode if input has value
  if (customVoltage.value) {
      updateCustomVoltage();
  } else if (power_vref_voltage.value !== 0) {
      // Pre-fill with current valid voltage if custom input is empty
      customVoltage.value = power_vref_voltage.value.toString();
      voltageErrorMsg.value = ""; // Clear any previous error
  } else {
      // Handle case where initial voltage is 0 (external) or invalid
      customVoltage.value = ""; // Keep it empty
      voltageErrorMsg.value = t('device_setting.voltage_errors.empty'); // Prompt user
  }
};

// 监听电压值变化，更新自定义输入框 (仅当非自定义模式或外部输入时)
watch(() => power_vref_voltage.value, (newVal) => {
  // Only update customVoltage if not currently in custom mode or if switching from external
  if (voltageMode.value !== 'custom' || newVal === 0) {
      if (newVal !== 0) {
          customVoltage.value = newVal.toString();
          voltageErrorMsg.value = ""; // Clear error when preset/external changes voltage
      } else {
          // Handle switching to external Vref
          customVoltage.value = ""; // Clear custom input if switching to external
          voltageErrorMsg.value = ""; // Clear error
      }
  }
}, { immediate: true });

// 处理复选框的复位模式切换
function toggleResetMode(mode: string, checked: boolean) {
  if (checked && !reset_mode.value.includes(mode)) {
    reset_mode.value.push(mode);
  } else if (!checked && reset_mode.value.includes(mode)) {
    const index = reset_mode.value.indexOf(mode);
    reset_mode.value.splice(index, 1);
  }
}

async function DownloadSetting() {
  console.log("download setting")
  let setting_str = JSON.stringify({
    name: "set_nickname",
    nickname: nickname.value
  })
  console.log(`setting str is ${setting_str}, len is ${setting_str.length}`)
  let rsp = await hslink_write_wait_rsp(setting_str, 1000)
  try {
    let rsp_json = JSON.parse(rsp)
    if (rsp_json["status"] == "success") {
      console.log("set nickname success")
    } else {
      alert_msg.value = t('device_setting.alert.fail')
      alert_type.value = "error"
      show_alert.value = true
      setTimeout(() => {
        show_alert.value = false
      }, 3000)
      console.log(`set nickname failed: ${rsp}`)
      return
    }
  } catch (e) {
    alert_msg.value = t('device_setting.alert.fail')
    alert_type.value = "error"
    show_alert.value = true
    setTimeout(() => {
      show_alert.value = false
    }, 3000)
    console.log(`download nickname failed: ${rsp}`)
    return
  }

  setting_str = JSON.stringify({
    name: "settings",
    data: {
      boost: speed_boost_enable.value,
      swd_port_mode: swd_simulate_mode.value,
      jtag_port_mode: jtag_simulate_mode.value,
      jtag_single_bit_mode: jtag_single_bit_mode.value,
      jtag_20pin_compatible: jtag_20pin_compatible.value,
      power: {
        power_on: power_power_on.value,
        port_on: power_port_on.value,
        vref: power_vref_voltage.value
      },
      reset: reset_mode.value,
      led: led_enable.value,
      led_brightness: led_brightness.value
    }
  })
  console.log(`setting str is ${setting_str}, len is ${setting_str.length}`)
  rsp = await hslink_write_wait_rsp(setting_str, 1000)

  try {
    let rsp_json = JSON.parse(rsp)
    if (rsp_json["status"] == "success") {
      alert_msg.value = t('device_setting.alert.success')
      alert_type.value = "success"
      show_alert.value = true
      setTimeout(() => {
        show_alert.value = false
      }, 3000)
      console.log("set setting success")
    } else {
      alert_msg.value = t('device_setting.alert.fail_reason') + rsp_json["message"]
      alert_type.value = "error"
      show_alert.value = true
      setTimeout(() => {
        show_alert.value = false
      }, 3000)
      console.log(`set setting failed: ${rsp}`)
    }
  } catch (e) {
    alert_msg.value = t('device_setting.alert.fail')
    alert_type.value = "error"
    show_alert.value = true
    setTimeout(() => {
      show_alert.value = false
    }, 3000)
    console.log(`download setting failed: ${rsp}`)
  }
}

</script>

<template>
  <div class="min-h-screen p-6 bg-base-200">
    <h2 class="text-2xl font-bold mb-6">{{ t('device_setting.title') }}</h2>
    
    <div v-if="!connected" class="alert alert-warning shadow-lg mb-6">
      <div>
        <span class="material-icons">warning</span>
        <span>{{ t('device_setting.connect_warning') }}</span>
      </div>
    </div>
    
    <div v-else class="space-y-6">
      <!-- 设备信息卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">device_hub</span>
            {{ t('device_setting.basic_info') }}
          </h2>
          <div class="form-control">
            <div class="relative">
              <div class="text-center mb-2">
                <span class="label-text text-lg font-medium">{{ t('device_info.name') }}</span>
              </div>
              <input type="text" :placeholder="t('device_setting.nickname_placeholder')" class="input input-bordered w-full" v-model="nickname"/>
            </div>
            <p class="text-xs mt-2 ml-2 opacity-70">{{ t('device_setting.nickname_desc') }}</p>
          </div>
        </div>
      </div>

      <!-- 性能设置卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">speed</span>
            {{ t('device_setting.performance') }}
          </h2>
          
          <div class="bg-base-200 p-3 rounded-lg">
            <div class="form-control">
              <div class="flex items-center justify-between">
                <span class="label-text text-lg">{{ t('device_setting.speed_boost') }}</span>
                <div class="flex items-center gap-2">
                  <input type="checkbox" class="toggle toggle-primary" 
                         :checked="speed_boost_enable" 
                         @change="speed_boost_enable = ($event.target as HTMLInputElement).checked"/>
                  <span class="label-text text-sm opacity-70">{{ speed_boost_enable ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
                </div>
              </div>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.speed_boost_desc') }}</p>
            </div>
          </div>
          
          <div class="divider my-2">{{ t('device_setting.interface_settings') }}</div>
          
          <div class="grid md:grid-cols-2 gap-6">
            <!-- 左侧：SWD输出方式 -->
            <div class="bg-base-200 p-4 rounded-lg">
              <label class="label justify-center">
                <span class="label-text text-lg font-medium">{{ t('device_setting.swd_output_mode') }}</span>
              </label>
              <div class="flex justify-center mt-2">
                <div class="flex gap-4">
                  <button class="btn px-6" 
                          :class="{'btn-primary': swd_simulate_mode === 'spi', 'btn-outline': swd_simulate_mode !== 'spi'}" 
                          @click="swd_simulate_mode = 'spi'">{{ t('device_setting.port_mode.spi') }}</button>
                  <button class="btn px-6"
                          :class="{'btn-primary': swd_simulate_mode === 'gpio', 'btn-outline': swd_simulate_mode !== 'gpio'}" 
                          @click="swd_simulate_mode = 'gpio'">{{ t('device_setting.port_mode.gpio') }}</button>
                </div>
              </div>
              <div class="text-xs text-center mt-3 opacity-70">
                <p>{{ t('device_setting.port_mode.spi_desc') }}</p>
                <p class="mt-1">{{ t('device_setting.port_mode.gpio_desc') }}</p>
              </div>
            </div>

            <!-- 右侧：JTAG输出方式 -->
            <div class="bg-base-200 p-4 rounded-lg">
              <label class="label justify-center">
                <span class="label-text text-lg font-medium">{{ t('device_setting.jtag_output_mode') }}</span>
              </label>
              <div class="flex justify-center mt-2">
                <div class="flex gap-4">
                  <button class="btn px-6" 
                          :class="{'btn-primary': jtag_simulate_mode === 'spi', 'btn-outline': jtag_simulate_mode !== 'spi'}" 
                          @click="jtag_simulate_mode = 'spi'">{{ t('device_setting.port_mode.spi') }}</button>
                  <button class="btn px-6"
                          :class="{'btn-primary': jtag_simulate_mode === 'gpio', 'btn-outline': jtag_simulate_mode !== 'gpio'}" 
                          @click="jtag_simulate_mode = 'gpio'">{{ t('device_setting.port_mode.gpio') }}</button>
                </div>
              </div>
              <div class="text-xs text-center mt-3 opacity-70">
                <p>{{ t('device_setting.port_mode.spi_desc') }}</p>
                <p class="mt-1">{{ t('device_setting.port_mode.gpio_desc') }}</p>
              </div>
            </div>
          </div>

          <!-- JTAG_SHIFT 加速选项 -->
          <div class="jtag-shift-container mt-4">
            <transition name="slide-fade">
              <div class="form-control" v-if="jtag_simulate_mode === 'spi'">
                <div class="bg-base-200 p-4 rounded-lg">
                  <div class="flex items-center justify-between">
                    <span class="label-text text-lg">{{ t('device_setting.jtag_shift') }}</span>
                    <div class="flex items-center gap-2">
                      <input type="checkbox" class="toggle toggle-primary" 
                             :checked="!jtag_single_bit_mode" 
                             @change="jtag_single_bit_mode = !($event.target as HTMLInputElement).checked"/>
                      <span class="label-text text-sm opacity-70">{{ !jtag_single_bit_mode ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
                    </div>
                  </div>
                  <div class="text-xs mt-1 ml-1 opacity-70">
                    {{ t('device_setting.jtag_shift_desc') }}
                  </div>
                </div>
              </div>
            </transition>
          </div>
        </div>
      </div>

      <!-- 兼容性选项卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">device_hub</span>
            {{ t('device_setting.compatibility') }}
          </h2>
          
          <div class="bg-base-200 p-4 rounded-lg">
            <div class="flex items-center justify-between">
              <span class="label-text text-lg">{{ t('device_setting.jtag_20pin') }}</span>
              <div class="flex items-center gap-2">
                <input type="checkbox" class="toggle toggle-primary" 
                       :checked="jtag_20pin_compatible" 
                       @change="jtag_20pin_compatible = ($event.target as HTMLInputElement).checked"/>
                <span class="label-text text-sm opacity-70">{{ jtag_20pin_compatible ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
              </div>
            </div>
            <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.jtag_20pin_desc') }}</p>
          </div>
        </div>
      </div>

      <!-- 电源设置卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">power</span>
            {{ t('device_setting.power_settings') }}
          </h2>
          
          <div class="grid md:grid-cols-2 gap-4">
            <!-- 左侧：上电开启电源输出-->
            <div class="form-control bg-base-200 p-3 rounded-lg">
              <div class="flex items-center justify-between">
                <span class="label-text text-lg">{{ t('device_setting.power_on_startup') }}</span>
                <div class="flex items-center gap-2">
                  <input type="checkbox" class="toggle toggle-primary" 
                         :checked="power_power_on" 
                         @change="power_power_on = ($event.target as HTMLInputElement).checked"/>
                  <span class="label-text text-sm opacity-70">{{ power_power_on ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
                </div>
              </div>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.power_on_desc') }}</p>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.power_on_desc2') }}</p>
            </div>
            
            <!-- 右侧：上电开启IO输出 -->
            <div class="form-control bg-base-200 p-3 rounded-lg">
              <div class="flex items-center justify-between">
                <span class="label-text text-lg">{{ t('device_setting.io_on_startup') }}</span>
                <div class="flex items-center gap-2">
                  <input type="checkbox" class="toggle toggle-primary" 
                         :checked="power_port_on" 
                         @change="power_port_on = ($event.target as HTMLInputElement).checked"/>
                  <span class="label-text text-sm opacity-70">{{ power_port_on ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
                </div>
              </div>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.io_on_desc') }}</p>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.io_on_desc2') }}</p>
            </div>
          </div>

          <div class="form-control mt-4">
            <div class="bg-base-200 p-4 rounded-lg">
              <div class="flex items-center justify-between mb-3">
                <span class="label-text text-lg font-medium">{{ t('device_setting.reference_voltage') }}</span>
                <div class="flex items-center gap-2">
                  <span class="text-sm">{{ t('device_setting.external_input') }}</span>
                  <input type="checkbox" class="toggle toggle-primary" 
                         :checked="isExternalVref" 
                         @change="isExternalVref = ($event.target as HTMLInputElement).checked"/>
                </div>
              </div>
              
              <transition
                name="fade"
                mode="out-in"
              >
                <div v-if="!isExternalVref" key="internal" class="space-y-3">
                  <!-- 电压设置模式选择 -->
                  <div class="tabs tabs-boxed">
                    <a class="tab" :class="{'tab-active': voltageMode === 'preset'}" @click="voltageMode = 'preset'">{{ t('device_setting.voltage_modes.preset') }}</a>
                    <a class="tab" :class="{'tab-active': voltageMode === 'custom'}" @click="switchToCustomMode">{{ t('device_setting.voltage_modes.custom') }}</a>
                  </div>
                  
                  <!-- 预设电压选择 -->
                  <transition name="fade" mode="out-in">
                    <div v-if="voltageMode === 'preset'" key="preset" class="grid grid-cols-4 gap-2">
                      <button v-for="voltage in presetVoltages" :key="voltage" 
                              class="btn"
                              :class="{'btn-primary': power_vref_voltage === voltage, 'btn-outline': power_vref_voltage !== voltage}"
                              @click="selectPresetVoltage(voltage)">
                        {{ voltage }}V
                      </button>
                    </div>
                    
                    <!-- 自定义电压输入 -->
                    <div v-else key="custom" class="form-control">
                      <div class="relative">
                        <div class="text-center mb-2 pb-1 border-b border-base-300 opacity-70">
                          <span class="text-sm font-medium">{{ t('device_setting.voltage_input') }}</span>
                        </div>
                        <div class="relative">
                          <input type="number"
                                 class="input input-bordered w-full transition-all duration-200 pr-8"
                                 :class="{
                                   'input-error': voltageErrorMsg,
                                   'focus:ring-2 focus:ring-error': voltageErrorMsg
                                 }"
                                 v-model="customVoltage"
                                 @input="updateCustomVoltage"
                                 @blur="updateCustomVoltage"
                                 step="0.1"
                                 min="1.8"
                                 max="5.0"
                                 :placeholder="t('device_setting.voltage_placeholder')"/>
                          <span class="absolute right-3 top-1/2 -translate-y-1/2 text-base-content opacity-70">V</span>
                        </div>
                      </div>
                      
                      <!-- Error message and help text -->
                      <div>
                        <transition name="fade">
                          <div v-if="voltageErrorMsg" 
                               class="text-error mt-2 text-sm font-medium bg-error/10 p-2 rounded-md border-l-4 border-error">
                            <div class="flex">
                              <span class="material-icons text-base mr-2">error_outline</span>
                              <div>
                                <strong>{{ t('device_setting.voltage_error') }}：</strong>{{ voltageErrorMsg }}
                                <div class="text-xs mt-1">
                                  <span v-if="voltageErrorMsg.includes('低于')">
                                    {{ t('device_setting.min_voltage') }}
                                  </span>
                                  <span v-else-if="voltageErrorMsg.includes('超过')">
                                    {{ t('device_setting.max_voltage') }}
                                  </span>
                                </div>
                              </div>
                            </div>
                          </div>
                          <div v-else class="text-xs mt-1 opacity-70 ml-1">{{ t('device_setting.voltage_range_desc') }}</div>
                        </transition>
                      </div>
                    </div>
                  </transition>
                  
                  <!-- Display current setting only if valid -->
                  <transition name="fade">
                    <div class="flex items-center" v-if="!voltageErrorMsg && !isExternalVref">
                      <span class="text-sm mr-2">{{ t('device_setting.current_setting') }}:</span>
                      <div class="badge badge-primary p-2">{{ power_vref_voltage }}V</div>
                    </div>
                  </transition>
                </div>
                
                <div v-else key="external" class="bg-base-100 p-4 rounded-lg text-center">
                  <div class="text-lg font-medium">{{ t('device_setting.external_vref_title') }}</div>
                  <p class="text-xs mt-1 opacity-70">{{ t('device_setting.external_vref_desc') }}</p>
                </div>
              </transition>
              
              <p class="text-xs mt-3 opacity-70">{{ t('device_setting.voltage_select_desc') }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- 复位设置卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">refresh</span>
            {{ t('device_setting.reset_settings') }}
          </h2>
          
          <div class="form-control">
            <div class="bg-base-200 p-4 rounded-lg">
              <label class="label">
                <span class="label-text text-lg font-medium">{{ t('device_setting.reset_modes') }}</span>
                <span class="text-xs opacity-70">{{ t('device_setting.reset_multi_select') }}</span>
              </label>
              <div class="grid md:grid-cols-3 gap-2 mt-2">
                <label class="flex items-center gap-2 p-3 bg-base-100 rounded-lg cursor-pointer border-2 transition-all hover:bg-base-300"
                       :class="{'border-primary': reset_mode.includes('nrst'), 'border-transparent': !reset_mode.includes('nrst')}">
                  <input type="checkbox" class="checkbox checkbox-primary" 
                         :checked="reset_mode.includes('nrst')"
                         @change="toggleResetMode('nrst', ($event.target as HTMLInputElement).checked)"/>
                  <div>
                    <div class="font-medium">{{ t('device_setting.reset_mode.nrst') }}</div>
                    <div class="text-xs opacity-70">{{ t('device_setting.reset_mode.nrst_desc') }}</div>
                  </div>
                </label>
                <label class="flex items-center gap-2 p-3 bg-base-100 rounded-lg cursor-pointer border-2 transition-all hover:bg-base-300" 
                       :class="{'border-primary': reset_mode.includes('por'), 'border-transparent': !reset_mode.includes('por')}">
                  <input type="checkbox" class="checkbox checkbox-primary" 
                         :checked="reset_mode.includes('por')"
                         @change="toggleResetMode('por', ($event.target as HTMLInputElement).checked)"/>
                  <div>
                    <div class="font-medium">{{ t('device_setting.reset_mode.por') }}</div>
                    <div class="text-xs opacity-70">{{ t('device_setting.reset_mode.por_desc') }}</div>
                  </div>
                </label>
                <label class="flex items-center gap-2 p-3 bg-base-100 rounded-lg cursor-pointer border-2 transition-all hover:bg-base-300" 
                       :class="{'border-primary': reset_mode.includes('arm_swd_soft'), 'border-transparent': !reset_mode.includes('arm_swd_soft')}">
                  <input type="checkbox" class="checkbox checkbox-primary" 
                         :checked="reset_mode.includes('arm_swd_soft')"
                         @change="toggleResetMode('arm_swd_soft', ($event.target as HTMLInputElement).checked)"/>
                  <div>
                    <div class="font-medium">{{ t('device_setting.reset_mode.arm_swd_soft') }}</div>
                    <div class="text-xs opacity-70">{{ t('device_setting.reset_mode.arm_swd_soft_desc') }}</div>
                  </div>
                </label>
              </div>
              <p class="text-xs mt-3 opacity-70">{{ t('device_setting.reset_desc') }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- LED设置卡 -->
      <div class="card bg-base-100 shadow-lg">
        <div class="card-body">
          <h2 class="card-title flex items-center">
            <span class="material-icons text-primary mr-2">lightbulb</span>
            {{ t('device_setting.led_settings') }}
          </h2>
          
          <div class="bg-base-200 p-4 rounded-lg">
            <div class="form-control">
              <div class="flex items-center justify-between">
                <span class="label-text text-lg">{{ t('device_setting.enable_led') }}</span>
                <div class="flex items-center gap-2">
                  <input type="checkbox" class="toggle toggle-primary" 
                         :checked="led_enable" 
                         @change="led_enable = ($event.target as HTMLInputElement).checked"/>
                  <span class="label-text text-sm opacity-70">{{ led_enable ? t('device_setting.enabled') : t('device_setting.disabled') }}</span>
                </div>
              </div>
              <p class="text-xs ml-1 mt-1 opacity-70">{{ t('device_setting.led_desc') }}</p>
            </div>
            
            <div class="led-brightness-container">
              <transition name="slide-fade">
                <div v-if="led_enable" class="form-control mt-4 led-brightness-content">
                  <label class="label">
                    <span class="label-text text-lg font-medium">{{ t('device_setting.led_brightness') }}</span>
                  </label>
                  <input type="range" min="1" max="100" v-model.number="led_brightness" class="range range-primary" step="1"/>
                  <div class="w-full flex justify-between text-xs px-2 mt-1">
                    <span>1%</span>
                    <span>25%</span>
                    <span>50%</span>
                    <span>75%</span>
                    <span>100%</span>
                  </div>
                  <div class="mt-2">
                    <div class="flex justify-between items-center">
                      <span class="text-sm opacity-70">{{ t('device_setting.current_brightness') }}:</span>
                      <div class="badge badge-primary p-3">{{ led_brightness }}%</div>
                    </div>
                  </div>
                  <p class="text-xs mt-3 opacity-70">{{ t('device_setting.led_brightness_desc') }}</p>
                </div>
              </transition>
            </div>
          </div>
        </div>
      </div>

      <!-- 保存按钮 -->
      <button class="btn btn-primary text-center w-full py-3 text-lg shadow-lg hover:scale-102 transition-all save-button" 
              @click="DownloadSetting" 
              :disabled="!canSave">
        <span class="material-icons mr-2">save</span>
        {{ t('device_setting.save_settings') }}
      </button>
    </div>
  </div>
  
  <!-- 提示组件 -->
  <transition
    enter-active-class="transition duration-300 ease-out"
    enter-from-class="transform translate-x-full opacity-0"
    enter-to-class="transform translate-x-0 opacity-100"
    leave-active-class="transition duration-200 ease-in"
    leave-from-class="transform translate-x-0 opacity-100"
    leave-to-class="transform translate-x-full opacity-0"
  >
    <div v-if="show_alert" 
         class="fixed top-4 right-4 p-4 rounded-lg shadow-lg flex items-center text-white z-50"
         :class="{'bg-success': alert_type === 'success', 'bg-error': alert_type === 'error'}">
      <span class="mr-2">
        <span class="material-icons" v-if="alert_type === 'success'">check_circle</span>
        <span class="material-icons" v-else>error</span>
      </span>
      <span>{{ alert_msg }}</span>
    </div>
  </transition>
</template>

<style scoped>
.transform {
  transform: translateX(0);
}

.translate-x-full {
  transform: translateX(100%);
}

.opacity-0 {
  opacity: 0;
}

.opacity-100 {
  opacity: 1;
}

.transition {
  transition-property: transform, opacity;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
}

.duration-200 {
  transition-duration: 200ms;
}

.duration-300 {
  transition-duration: 300ms;
}

.ease-in {
  transition-timing-function: cubic-bezier(0.4, 0, 1, 1);
}

.ease-out {
  transition-timing-function: cubic-bezier(0, 0, 0.2, 1);
}

/* Optional: Add slightly more emphasis to the error input */
.input-error {
  border-color: hsl(var(--er) / var(--tw-border-opacity));
  --tw-border-opacity: 1;
  border-width: 2px; /* Make border thicker on error */
}

/* Ensure error icon size is appropriate */
.material-icons.text-base {
  font-size: 1.125rem; /* Adjust size as needed */
  line-height: 1; /* Align icon better */
}

/* 隐藏数字输入框的上下调节按钮 */
input[type="number"]::-webkit-inner-spin-button,
input[type="number"]::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

input[type="number"] {
  -moz-appearance: textfield; /* Firefox */
}

/* 禁止文本选中和复制 */
.min-h-screen {
  user-select: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

/* 确保输入框内的文本仍可以选择和编辑 */
input, textarea {
  user-select: text;
  -webkit-user-select: text;
  -moz-user-select: text;
  -ms-user-select: text;
}

/* 保存按钮样式 */
.save-button {
  position: relative;
  height: 3rem !important; /* 降低按钮高度 */
  padding-top: 0.5rem !important;
  padding-bottom: 0.5rem !important;
}

.save-button .material-icons {
  margin-right: 0.25rem; /* 图标与文字间距缩小 */
}

/* 悬停效果 */
.hover\:scale-102:hover {
  transform: scale(1.02); /* 减小悬停放大效果 */
}

/* 过渡动画 - 展开/收起 */
.expand-enter-active,
.expand-leave-active {
  transition: height 0.3s ease, opacity 0.3s ease;
  overflow: hidden;
}

.expand-enter-from,
.expand-leave-to {
  height: 0;
  opacity: 0;
}

/* 过渡动画 - 淡入淡出 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 过渡动画 - 滑动淡入淡出 */
.slide-fade-enter-active,
.slide-fade-leave-active {
  transition: all 0.3s ease;
  overflow: hidden;
  max-height: 300px; /* 足够容纳内容的最大高度 */
}

.slide-fade-enter-from,
.slide-fade-leave-to {
  max-height: 0;
  opacity: 0;
  margin-top: 0;
  padding-top: 0;
  padding-bottom: 0;
}

/* LED亮度容器样式 */
.led-brightness-container {
  position: relative;
}

.led-brightness-content {
  transform-origin: top;
}
</style>