<template>
  <view :class="classes" @click="onClick" :style="style">
    <view class="switch-button">
      <nut-icon v-if="loading" :name="name" :size="size" :color="color"></nut-icon>
      <!-- <view v-show="!modelValue" class="close-line"></view> -->
      <template v-if="activeText">
        <view class="nut-switch-label open" v-show="modelValue">{{ activeText }}</view>
        <view class="nut-switch-label close" v-show="!modelValue">{{ inactiveText }}</view>
      </template>
    </view>
  </view>
</template>

<script lang="ts">
import { computed } from 'vue';
import { createComponent } from '../../utils/create';
const { componentName, create } = createComponent('switch');

export default create({
  props: {
    modelValue: {
      type: [String, Number, Boolean],
      default: false
    },
    disable: {
      type: Boolean,
      default: false
    },
    activeColor: {
      type: String,
      default: ''
    },
    inactiveColor: {
      type: String,
      default: ''
    },
    activeText: {
      type: String,
      default: ''
    },
    inactiveText: {
      type: String,
      default: ''
    },
    activeValue: {
      type: [String, Number, Boolean],
      default: true
    },
    inactiveValue: {
      type: [String, Number, Boolean],
      default: false
    },
    loading: {
      type: Boolean,
      default: false
    },
    name: {
      type: String,
      default: 'loading1'
    },
    size: {
      type: [String, Number],
      default: '12px'
    },
    color: {
      type: String,
      default: ''
    }
  },
  emits: ['change', 'update:modelValue'],
  setup(props, { emit }) {
    const isActive = computed(() => props.modelValue === props.activeValue);
    const classes = computed(() => {
      const prefixCls = componentName;
      return {
        [prefixCls]: true,
        [isActive.value ? 'switch-open' : 'switch-close']: true,
        [`${prefixCls}-disable`]: props.disable,
        [`${prefixCls}-base`]: true
      };
    });

    const style = computed(() => {
      return {
        backgroundColor: isActive.value ? props.activeColor : props.inactiveColor
      };
    });

    const onClick = (event: Event) => {
      if (props.disable || props.loading) return;
      const value = isActive.value ? props.inactiveValue : props.activeValue;
      emit('update:modelValue', value);
      emit('update:loading');
      emit('change', value, event);
    };

    return {
      classes,
      style,
      onClick
    };
  }
});
</script>
