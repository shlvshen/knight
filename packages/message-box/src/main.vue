<template>
  <transition name="msgbox-fade">
    <div class="el-message-box__wrapper" tabindex="-1" v-show="visible" @click.self="handleWrapperClick">
      <div class="el-message-box" :class="customClass">
        <div class="el-message-box__header" v-if="title !== undefined">
          <div class="el-message-box__title">{{ title }}</div>
          <button type="button" class="el-message-box__headerbtn" aria-label="Close" 
                  v-if="showClose" @click="handleAction('cancel')">
            <i class="el-message-box__close el-icon-close"></i>
          </button>
        </div>
        <div class="el-message-box__content" v-if="message !== ''">
          <div class="el-message-box__message" :style="{ 'text-align': typeClass ? 'center' : 'initial' }">
            <p>
              <span class="el-message-box__status iconfont" :class="[ type ]" v-html="typeClass"></span>
              <slot><span :style="{'padding-left': typeClass ? '46px' : '0px'}">{{ message }}</span></slot>
            </p>
          </div>
          <div class="el-message-box__input" v-show="showInput">
            <el-input v-model="inputValue" @keyup.enter.native="handleAction('confirm')" :placeholder="inputPlaceholder" ref="input"></el-input>
            <div class="el-message-box__errormsg" :style="{ visibility: !!editorErrorMessage ? 'visible' : 'hidden' }">{{ editorErrorMessage }}</div>
          </div>
        </div>
        <div class="el-message-box__btns">
          <el-button
            :loading="confirmButtonLoading"
            ref="confirm"
            size="small"
            :class="[ confirmButtonClasses ]"
            v-show="showConfirmButton"
            @click.native="handleAction('confirm')">
            {{ confirmButtonText || t('el.messagebox.confirm') }}
          </el-button>
          <el-button
            :loading="cancelButtonLoading"
            :class="[ cancelButtonClasses ]"
            v-show="showCancelButton"
            size="small"
            type="cancel"
            @click.native="handleAction('cancel')">
            {{ cancelButtonText || t('el.messagebox.cancel') }}
          </el-button>
        </div>
      </div>
    </div>
  </transition>
</template>

<script type="text/babel">
  import Popup from 'thx-knight/src/utils/popup';
  import Locale from 'thx-knight/src/mixins/locale';
  import ElInput from 'thx-knight/packages/input';
  import ElButton from 'thx-knight/packages/button';
  import { addClass, removeClass } from 'thx-knight/src/utils/dom';
  import { t } from 'thx-knight/src/locale';

  // let typeMap = {
  //   success: 'circle-check',
  //   info: 'information',
  //   warning: 'warning',
  //   error: 'circle-cross'
  // };
  let typeMap = {
    success: '&#xe77f;',
    info: '&#xe659;',
    warning: '&#xe65a;',
    danger: '&#xe695;'
  };

  export default {
    mixins: [Popup, Locale],

    props: {
      modal: {
        default: true
      },
      lockScroll: {
        default: true
      },
      showClose: {
        type: Boolean,
        default: true
      },
      closeOnClickModal: {
        default: true
      },
      closeOnPressEscape: {
        default: true
      }
    },

    components: {
      ElInput,
      ElButton
    },

    computed: {
      typeClass() {
        return this.type && typeMap[this.type] ? `${ typeMap[this.type] }` : '';
        // return this.type && typeMap[this.type] ? `el-icon-${ typeMap[this.type] }` : '';
      },

      confirmButtonClasses() {
        return `el-button--primary ${ this.confirmButtonClass }`;
      },
      cancelButtonClasses() {
        return `${ this.cancelButtonClass }`;
      }
    },

    methods: {
      getSafeClose() {
        const currentId = this.uid;
        return () => {
          this.$nextTick(() => {
            if (currentId === this.uid) this.doClose();
          });
        };
      },
      doClose() {
        if (!this.visible) return;
        this.visible = false;
        this._closing = true;

        this.onClose && this.onClose();

        if (this.lockScroll) {
          setTimeout(() => {
            if (this.modal && this.bodyOverflow !== 'hidden') {
              document.body.style.overflow = this.bodyOverflow;
              document.body.style.paddingRight = this.bodyPaddingRight;
            }
            this.bodyOverflow = null;
            this.bodyPaddingRight = null;
          }, 200);
        }
        this.opened = false;

        if (!this.transition) {
          this.doAfterClose();
        }
        if (this.action) this.callback(this.action, this);
      },

      handleWrapperClick() {
        if (this.closeOnClickModal) {
          this.handleAction('cancel');
        }
      },

      handleAction(action) {
        if (this.$type === 'prompt' && action === 'confirm' && !this.validate()) {
          return;
        }
        this.action = action;
        if (typeof this.beforeClose === 'function') {
          this.close = this.getSafeClose();
          this.beforeClose(action, this, this.close);
        } else {
          this.doClose();
        }
      },

      validate() {
        if (this.$type === 'prompt') {
          var inputPattern = this.inputPattern;
          if (inputPattern && !inputPattern.test(this.inputValue || '')) {
            this.editorErrorMessage = this.inputErrorMessage || t('el.messagebox.error');
            addClass(this.$refs.input.$el.querySelector('input'), 'invalid');
            return false;
          }
          var inputValidator = this.inputValidator;
          if (typeof inputValidator === 'function') {
            var validateResult = inputValidator(this.inputValue);
            if (validateResult === false) {
              this.editorErrorMessage = this.inputErrorMessage || t('el.messagebox.error');
              addClass(this.$refs.input.$el.querySelector('input'), 'invalid');
              return false;
            }
            if (typeof validateResult === 'string') {
              this.editorErrorMessage = validateResult;
              return false;
            }
          }
        }
        this.editorErrorMessage = '';
        removeClass(this.$refs.input.$el.querySelector('input'), 'invalid');
        return true;
      }
    },

    watch: {
      inputValue: {
        immediate: true,
        handler(val) {
          this.$nextTick(_ => {
            if (this.$type === 'prompt' && val !== null) {
              this.validate();
            }
          });
        }
      },

      visible(val) {
        if (val) this.uid++;
        if (this.$type === 'alert' || this.$type === 'confirm') {
          this.$nextTick(() => {
            this.$refs.confirm.$el.focus();
          });
        }
        if (this.$type !== 'prompt') return;
        if (val) {
          setTimeout(() => {
            if (this.$refs.input && this.$refs.input.$el) {
              this.$refs.input.$el.querySelector('input').focus();
            }
          }, 500);
        } else {
          this.editorErrorMessage = '';
          removeClass(this.$refs.input.$el.querySelector('input'), 'invalid');
        }
      }
    },

    data() {
      return {
        uid: 1,
        title: undefined,
        message: '',
        type: '',
        customClass: '',
        showInput: false,
        inputValue: null,
        inputPlaceholder: '',
        inputPattern: null,
        inputValidator: null,
        inputErrorMessage: '',
        showConfirmButton: true,
        showCancelButton: false,
        action: '',
        confirmButtonText: '',
        cancelButtonText: '',
        confirmButtonLoading: false,
        cancelButtonLoading: false,
        confirmButtonClass: '',
        confirmButtonDisabled: false,
        cancelButtonClass: '',
        editorErrorMessage: null,
        callback: null
      };
    }
  };
</script>
<style type="scss">
  @font-face {
    font-family: 'iconfont';  /* project id 313336 */
    src: url('//at.alicdn.com/t/font_313336_5ilrj33g5b2ihpvi.eot');
    src: url('//at.alicdn.com/t/font_313336_5ilrj33g5b2ihpvi.eot?#iefix') format('embedded-opentype'),
    url('//at.alicdn.com/t/font_313336_5ilrj33g5b2ihpvi.woff') format('woff'),
    url('//at.alicdn.com/t/font_313336_5ilrj33g5b2ihpvi.ttf') format('truetype'),
    url('//at.alicdn.com/t/font_313336_5ilrj33g5b2ihpvi.svg#iconfont') format('svg');
  }
  .iconfont{
    font-family:"iconfont" !important;
    font-size:16px;font-style:normal;
    -webkit-font-smoothing: antialiased;
    -webkit-text-stroke-width: 0.2px;
    -moz-osx-font-smoothing: grayscale;
    /* font-size: 18px; */
  }
</style>
