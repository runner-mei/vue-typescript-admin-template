<template>
  <div
    :class="{fullscreen:fullscreen}"
    class="tinymce-container editor-container"
  >
    <textarea
      :id="tinymceId"
      class="tinymce-textarea"
    />
    <div class="editor-custom-btn-container">
      <editorImage
        color="#1890ff"
        class="editor-upload-btn"
        @successCBK="imageSuccessCBK"
      />
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Vue, Prop, Watch } from 'vue-property-decorator';
import editorImage from './components/editorImage.vue';
import plugins from './plugins';
import toolbar from './toolbar';

@Component({
  components: {
    editorImage,
  },
})
export default class Tinymce extends Vue {
  @Prop({ default: () => 'vue-tinymce-' + +new Date() + ((Math.random() * 1000).toFixed(0) + '') })
  id!: string;

  @Prop({ default: '' })
  value!: string;

  @Prop({
    required: false,
    default: () => [],
  })
  toolbar!: Array<any>;

  @Prop({ default: 'file edit insert view format table' })
  menubar!: string;

  @Prop({ required: false, default: 360 })
  height!: number;

  private hasChange: boolean = false;
  private hasInit: boolean = false;
  private tinymceId: string = this.id;
  private fullscreen: boolean = false;
  private languageTypeList: { [key: string]: string } ={
    'en': 'en',
    'zh': 'zh_CN',
  };

  get language(): string {
    return this.languageTypeList[this.$store.getters.language];
  }

  @Watch('value')
  onValue(val: string) {
    if (!this.hasChange && this.hasInit) {
      this.$nextTick(() =>
        (window as any).tinymce.get(this.tinymceId).setContent(val || ''));
    }
  }

  @Watch('language')
  onLanguage() {
    this.destroyTinymce();
    this.$nextTick(() => this.initTinymce());
  }

  mounted() {
    this.initTinymce();
  }
  activated() {
    this.initTinymce();
  }
  deactivated() {
    this.destroyTinymce();
  }
  destroyed() {
    this.destroyTinymce();
  }

  initTinymce() {
    const _this = this;
    (window as any).tinymce.init({
      language: this.language,
      selector: `#${this.tinymceId}`,
      height: this.height,
      body_class: 'panel-body ',
      object_resizing: false,
      toolbar: this.toolbar.length > 0 ? this.toolbar : toolbar,
      menubar: this.menubar,
      plugins: plugins,
      end_container_on_empty_block: true,
      powerpaste_word_import: 'clean',
      code_dialog_height: 450,
      code_dialog_width: 1000,
      advlist_bullet_styles: 'square',
      advlist_number_styles: 'default',
      imagetools_cors_hosts: ['www.tinymce.com', 'codepen.io'],
      default_link_target: '_blank',
      link_title: false,
      nonbreaking_force_tab: true, // inserting nonbreaking space &nbsp; need Nonbreaking Space Plugin
      init_instance_callback: (editor: any) => {
        if (_this.value) {
          editor.setContent(_this.value);
        }
        _this.hasInit = true;
        editor.on('NodeChange Change KeyUp SetContent', () => {
          this.hasChange = true;
          this.$emit('input', editor.getContent());
        });
      },
      setup(editor: any) {
        editor.on('FullscreenStateChanged', (e: any) => {
          _this.fullscreen = e.state;
        });
      },
      // 整合七牛上传
      // images_dataimg_filter(img) {
      //   setTimeout(() => {
      //     const $image = $(img);
      //     $image.removeAttr('width');
      //     $image.removeAttr('height');
      //     if ($image[0].height && $image[0].width) {
      //       $image.attr('data-wscntype', 'image');
      //       $image.attr('data-wscnh', $image[0].height);
      //       $image.attr('data-wscnw', $image[0].width);
      //       $image.addClass('wscnph');
      //     }
      //   }, 0);
      //   return img
      // },
      // images_upload_handler(blobInfo, success, failure, progress) {
      //   progress(0);
      //   const token = _this.$store.getters.token;
      //   getToken(token).then(response => {
      //     const url = response.data.qiniu_url;
      //     const formData = new FormData();
      //     formData.append('token', response.data.qiniu_token);
      //     formData.append('key', response.data.qiniu_key);
      //     formData.append('file', blobInfo.blob(), url);
      //     upload(formData).then(() => {
      //       success(url);
      //       progress(100);
      //     })
      //   }).catch(err => {
      //     failure('出现未知问题，刷新页面，或者联系程序员')
      //     console.log(err);
      //   });
      // },
    });
  }
  destroyTinymce() {
    const tinymce = (window as any).tinymce.get(this.tinymceId);
    if (this.fullscreen) {
      tinymce.execCommand('mceFullScreen');
    }

    if (tinymce) {
      tinymce.destroy();
    }
  }
  setContent(value: any) {
    (window as any).tinymce.get(this.tinymceId).setContent(value);
  }
  getContent() {
    (window as any).tinymce.get(this.tinymceId).getContent();
  }
  imageSuccessCBK(arr: any[]) {
    const _this = this;
    arr.forEach((v: any) => {
      (window as any).tinymce.get(_this.tinymceId).insertContent(`<img class="wscnph" src="${v.url}" >`);
    });
  }
}
</script>

<style scoped>
.tinymce-container {
  position: relative;
  line-height: normal;
}
.tinymce-container>>>.mce-fullscreen {
  z-index: 10000;
}
.tinymce-textarea {
  visibility: hidden;
  z-index: -1;
}
.editor-custom-btn-container {
  position: absolute;
  right: 4px;
  top: 4px;
  /*z-index: 2005;*/
}
.fullscreen .editor-custom-btn-container {
  z-index: 10000;
  position: fixed;
}
.editor-upload-btn {
  display: inline-block;
}
</style>
