<template>
  <view :class="classes">
    <view class="nut-uploader__slot" v-if="$slots.default">
      <slot></slot>
      <template v-if="maximum - fileList.length">
        <input
          class="nut-uploader__input"
          v-if="capture"
          type="file"
          capture="camera"
          :accept="accept"
          :multiple="multiple"
          :name="name"
          :disabled="disabled"
          @change="onChange"
        />
        <input
          class="nut-uploader__input"
          v-else
          type="file"
          :accept="accept"
          :multiple="multiple"
          :name="name"
          :disabled="disabled"
          @change="onChange"
        />
      </template>
    </view>

    <view class="nut-uploader__preview" :class="[listType]" v-for="(item, index) in fileList" :key="item.uid">
      <view class="nut-uploader__preview-img" v-if="listType == 'picture' && !$slots.default">
        <view class="nut-uploader__preview__progress" v-if="item.status == 'ready'">
          <view class="nut-uploader__preview__progress__msg">{{ item.message }}</view>
        </view>
        <view class="nut-uploader__preview__progress" v-else-if="item.status != 'success'">
          <nut-icon color="#fff" :name="item.status == 'error' ? 'failure' : 'loading'"></nut-icon>
          <view class="nut-uploader__preview__progress__msg">{{ item.message }}</view>
        </view>
        <nut-icon
          v-if="isDeletable"
          color="rgba(0,0,0,0.6)"
          @click="onDelete(item, index)"
          class="close"
          name="failure"
        ></nut-icon>
        <img
          class="nut-uploader__preview-img__c"
          @click="fileItemClick(item)"
          v-if="item.type.includes('image') && item.url"
          :src="item.url"
        />
        <view v-else class="nut-uploader__preview-img__file">
          <view @click="fileItemClick(item)" class="nut-uploader__preview-img__file__name">
            <nut-icon color="#808080" name="link"></nut-icon>&nbsp;{{ item.name }}
          </view>
        </view>
        <view class="tips">{{ item.name }}</view>
      </view>
      <view class="nut-uploader__preview-list" v-else-if="listType == 'list'">
        <view @click="fileItemClick(item)" class="nut-uploader__preview-img__file__name" :class="[item.status]">
          <nut-icon name="link" />&nbsp;{{ item.name }}
        </view>
        <nut-icon
          class="nut-uploader__preview-img__file__del"
          @click="onDelete(item, index)"
          color="#808080"
          name="del"
        />
        <nut-progress
          size="small"
          :percentage="item.percentage"
          v-if="item.status == 'uploading'"
          stroke-color="linear-gradient(270deg, rgba(18,126,255,1) 0%,rgba(32,147,255,1) 32.815625%,rgba(13,242,204,1) 100%)"
          :show-text="false"
        >
        </nut-progress>
      </view>
    </view>
    <view
      class="nut-uploader__upload"
      :class="[listType]"
      v-if="listType == 'picture' && !$slots.default && maximum - fileList.length"
    >
      <nut-icon :size="uploadIconSize" color="#808080" :name="uploadIcon"></nut-icon>
      <input
        class="nut-uploader__input"
        v-if="capture"
        type="file"
        capture="camera"
        :accept="accept"
        :multiple="multiple"
        :name="name"
        :disabled="disabled"
        @change="onChange"
      />
      <input
        class="nut-uploader__input"
        v-else
        type="file"
        :accept="accept"
        :multiple="multiple"
        :name="name"
        :disabled="disabled"
        @change="onChange"
      />
    </view>
  </view>
</template>

<script lang="ts">
import { computed, reactive } from 'vue';
import { createComponent } from '../../utils/create';
import { Uploader, UploadOptions } from './uploader';
const { componentName, create } = createComponent('uploader');
export type FileItemStatus = 'ready' | 'uploading' | 'success' | 'error';
export class FileItem {
  status: FileItemStatus = 'ready';
  message: string = '准备完成';
  uid: string = new Date().getTime().toString();
  name?: string;
  url?: string;
  type?: string;
  percentage: string | number = 0;
  formData: FormData = new FormData();
}
export default create({
  props: {
    name: { type: String, default: 'file' },
    url: { type: String, default: '' },
    // defaultFileList: { type: Array, default: () => new Array<FileItem>() },
    timeout: { type: [Number, String], default: 1000 * 30 },
    fileList: { type: Array, default: () => [] },
    isPreview: { type: Boolean, default: true },
    // picture、list
    listType: { type: String, default: 'picture' },
    isDeletable: { type: Boolean, default: true },
    method: { type: String, default: 'post' },
    capture: { type: Boolean, default: false },
    maximize: { type: [Number, String], default: Number.MAX_VALUE },
    maximum: { type: [Number, String], default: 1 },
    clearInput: { type: Boolean, default: true },
    accept: { type: String, default: '*' },
    headers: { type: Object, default: {} },
    data: { type: Object, default: {} },
    uploadIcon: { type: String, default: 'photograph' },
    uploadIconSize: { type: [String, Number], default: '' },
    xhrState: { type: [Number, String], default: 200 },
    withCredentials: { type: Boolean, default: false },
    multiple: { type: Boolean, default: false },
    disabled: { type: Boolean, default: false },
    autoUpload: { type: Boolean, default: true },
    beforeUpload: {
      type: Function,
      default: null
    },
    beforeDelete: {
      type: Function,
      default: (file: FileItem, files: FileItem[]) => {
        return true;
      }
    },
    onChange: { type: Function }
    // customRequest: { type: Function }
  },
  emits: [
    'start',
    'progress',
    'oversize',
    'success',
    'failure',
    'change',
    'delete',
    'update:fileList',
    'file-item-click'
  ],
  setup(props, { emit }) {
    const fileList = reactive(props.fileList) as Array<FileItem>;
    let uploadQueue: Promise<Uploader>[] = [];

    const classes = computed(() => {
      const prefixCls = componentName;
      return {
        [prefixCls]: true
      };
    });

    const clearInput = (el: HTMLInputElement) => {
      el.value = '';
    };

    const fileItemClick = (fileItem: FileItem) => {
      emit('file-item-click', { fileItem });
    };

    const executeUpload = (fileItem: FileItem, index: number) => {
      const uploadOption = new UploadOptions();
      uploadOption.url = props.url;
      uploadOption.formData = fileItem.formData;
      uploadOption.timeout = (props.timeout as number) * 1;
      uploadOption.method = props.method;
      uploadOption.xhrState = props.xhrState as number;
      uploadOption.headers = props.headers;
      uploadOption.withCredentials = props.withCredentials;
      uploadOption.onStart = (option: UploadOptions) => {
        fileItem.status = 'ready';
        fileItem.message = '准备上传';
        clearUploadQueue(index);
        emit('start', option);
      };
      uploadOption.onProgress = (event: ProgressEvent<XMLHttpRequestEventTarget>, option: UploadOptions) => {
        fileItem.status = 'uploading';
        fileItem.message = '上传中';
        fileItem.percentage = ((event.loaded / event.total) * 100).toFixed(0);
        emit('progress', { event, option, percentage: fileItem.percentage });
      };

      uploadOption.onSuccess = (responseText: XMLHttpRequest['responseText'], option: UploadOptions) => {
        fileItem.status = 'success';
        fileItem.message = '上传成功';
        emit('success', {
          responseText,
          option,
          fileItem
        });
        emit('update:fileList', fileList);
      };
      uploadOption.onFailure = (responseText: XMLHttpRequest['responseText'], option: UploadOptions) => {
        fileItem.status = 'error';
        fileItem.message = '上传失败';
        emit('failure', {
          responseText,
          option,
          fileItem
        });
      };
      let task = new Uploader(uploadOption);
      if (props.autoUpload) {
        task.upload();
      } else {
        uploadQueue.push(
          new Promise((resolve, reject) => {
            resolve(task);
          })
        );
      }
    };
    const clearUploadQueue = (index = -1) => {
      if (index > -1) {
        uploadQueue.splice(index, 1);
      } else {
        uploadQueue = [];
      }
    };
    const submit = () => {
      Promise.all(uploadQueue).then((res) => {
        res.forEach((i) => i.upload());
      });
    };

    const readFile = (files: File[]) => {
      files.forEach((file: File, index: number) => {
        const formData = new FormData();
        for (const [key, value] of Object.entries(props.data)) {
          formData.append(key, value);
        }
        formData.append(props.name, file);

        const fileItem = reactive(new FileItem());
        fileItem.name = file.name;
        fileItem.status = 'ready';
        fileItem.type = file.type;
        fileItem.formData = formData;
        fileItem.message = '等待上传';
        executeUpload(fileItem, index);

        if (props.isPreview && file.type.includes('image')) {
          const reader = new FileReader();
          reader.onload = (event: ProgressEvent<FileReader>) => {
            fileItem.url = (event.target as FileReader).result as string;
            fileList.push(fileItem);
          };
          reader.readAsDataURL(file);
        } else {
          fileList.push(fileItem);
        }
      });
    };

    const filterFiles = (files: File[]) => {
      const maximum = (props.maximum as number) * 1;
      const maximize = (props.maximize as number) * 1;
      const oversizes = new Array<File>();
      files = files.filter((file: File) => {
        if (file.size > maximize) {
          oversizes.push(file);
          return false;
        } else {
          return true;
        }
      });
      if (oversizes.length) {
        emit('oversize', oversizes);
      }
      if (files.length > maximum) {
        files.splice(maximum - 1, files.length - maximum);
      }
      return files;
    };
    const onDelete = (file: FileItem, index: number) => {
      clearUploadQueue(index);
      if (props.beforeDelete(file, fileList)) {
        fileList.splice(index, 1);
        emit('delete', {
          file,
          fileList
        });
      } else {
        console.log('用户阻止了删除！');
      }
    };

    const onChange = (event: InputEvent) => {
      if (props.disabled) {
        return;
      }
      const $el = event.target as HTMLInputElement;
      let { files } = $el;

      if (props.beforeUpload) {
        props.beforeUpload(files).then((f: Array<File>) => {
          const _files: File[] = filterFiles(new Array<File>().slice.call(f));
          readFile(_files);
        });
      } else {
        const _files: File[] = filterFiles(new Array<File>().slice.call(files));
        readFile(_files);
      }

      emit('change', {
        fileList,
        event
      });

      if (props.clearInput) {
        clearInput($el);
      }
    };

    return {
      onChange,
      onDelete,
      fileList,
      classes,
      fileItemClick,
      clearUploadQueue,
      submit
    };
  }
});
</script>
