<template>
  <div :class="classes" @click="select" v-el:select>
    <div class="ant-select-selection ant-select-selection--single" :style="css">
      <div class="ant-select-selection__rendered">
        <div v-if="!selectedVal && placeholder" v-show="placeholder_show" class="ant-select-selection__placeholder"
             style="display: block; -webkit-user-select: none;">{{ placeholder }}
        </div>
        <div v-else class="ant-select-selection-selected-value" v-show="value_show" :style="value_opacity">
          <slot></slot>
          {{selectedVal}}
        </div>
        <div v-if="type=='search'" class="ant-select-search ant-select-search--inline" v-show="selected">
          <div class="ant-select-search__field__wrap">
            <input @input="keyat" v-on:focus="value_opacity.opacity=.4" v-on:blur="value_opacity.opacity=1" class="ant-select-search__field" v-el:searchinput>
            <span class="ant-select-search__field__mirror"></span>
          </div>
        </div>
      </div>
      <span unselectable="unselectable" class="ant-select-selection__clear" style="-webkit-user-select: none" v-if="allowClear && !multiple && value" @click.stop="clear"></span>
      <span class="ant-select-arrow" style="-webkit-user-select: none;"><b></b></span>
    </div>
    <X-Option v-if="!disabled" :stylus.sync="stylus" :disabled="disabled" :show.sync="selected" :options.sync="options"
    :class="clazz" :multiple="multiple" :placeholder="placeholder" :notfound="notfound" :value="value" :position="position" v-el:dropdownlist></X-Option>
  </div>
</template>
<script>
  import XOption from './option.vue'
  import {getOffset, closeByElement} from '../_util/_func'

  export default {
    name: 'v-select',
    data:()=>({
        stylus: {
          top: 0,
          left: 0,
          width:null,
          height:null
        },
        value_opacity: {
          opacity: '1'
        },
        origin_placeholder: ''
    }),
    props: {
      type: String,
      id: String,
      selected: Boolean,
      size: String,
      disabled: Boolean,
      allowClear: {
        type: Boolean,
        default: true
      },
      options: {
        type: Array,
        default(){
          return []
        }
      },
      value: [String,Number],
      multiple: Boolean,
      placeholder: String,
      placeholder_show: {
        type: Boolean,
        default: true
      },
      value_show: {
        type: Boolean,
        default: true
      },
      notfound: String,
      position: {
        type: String,
        default: "bottom"
      },
    },
    computed: {
      classes () {
        return [
          'ant-select',
          this.disabled ? 'ant-select-disabled' : 'ant-select-enabled',
          {
            'ant-select-open': this.selected,
            'ant-select-focused': this.selected,
            'ant-select-allow-clear': this.allowClear && !this.multiple,
            [`ant-select-${this.size}`]: this.size,
            'ant-select-show-search': (this.type === 'search')
          }
        ]
      },
      clazz () {
        return [{
          'ant-select-dropdown--multiple': this.multiple
        }]
      },
      css () {
        if (this.type === 'search' && this.placeholder) {
          return {
            cursor: 'text'
          }
        }else if(this.disabled){
          return {
            cursor: 'not-allowed'
          }
        }
        return {
          cursor: 'pointer'
        }
      },
      source(){
        //如果search下拉,则保存原始数据源，删选后回退用
        return this.type === 'search'? [...this.options] : this.options
      },
      selectedVal(){
        if (this.source instanceof Array && this.source.length) {
          if(this.value){
            for(let item of this.source){
              if(item.value == this.value){
                return item.text
              }
            }
          }else if(!this.origin_placeholder){
            this.value = this.source[0].value
            return this.source[0].text
          }
        }
      }
    },
    watch: {
      value(val, oldVal){
        if(val){
          this.placeholder = ''
        }else if(this.origin_placeholder){
          this.placeholder = this.origin_placeholder;
        }
      }
    },
    methods: {
      clear(){
        this.value = ''
        this.$dispatch('select-change', {value:'',text:''})
      },
      select () {
        if (!this.disabled) {
          this.setPosition();
          this.selected = !this.selected
          if (this.type === 'search' && this.selected) {
            var that = this
            this.$els.searchinput.value = ''
            this.options = this.source
            setTimeout(function () {
              that.$els.searchinput.focus()
            }, 50)
          }
        }
      },
      setPosition () {
        let p = getOffset(this.$els.select)
        this.stylus = Object.assign({},this.stylus,{
          top: p.top,
          left: p.left,
          width: this.stylus.width,
          height:this.height
        });
      },
      backdrop (e) {
        if (!closeByElement(e.target, [this.$els.select, this.$els.dropdownlist])) {
          this.selected = false
          if (this.placeholder) {
            this.placeholder_show = true
          } else {
            this.value_show = true
          }
        }
      },
      keyat (e) {
//        只有当type==search 的时候才会有搜索框
        let keyword = this.$els.searchinput.value
        if (this.placeholder) {
          if (keyword.length > 0) {
            this.placeholder_show = false
          } else {
            this.placeholder_show = true
          }
        } else {
          if (keyword.length > 0) {
            this.value_show = false
          } else {
            this.value_show = true
          }
        }
        let word = new RegExp(keyword)
        let newlist = []
        for (let i = 0; i < this.source.length; i++) {
          if (word.test(this.source[i].text)) {
            newlist.push(Object.assign({}, this.source[i]))
          }
        }
        this.options = newlist
      }
    },
    created: function () {
      document.addEventListener('click', this.backdrop)
    },
    ready: function () {
      this.origin_placeholder = this.placeholder
      let that = this
      let styles = window.getComputedStyle(this.$els.select)
      this.height = parseFloat(styles.getPropertyValue('height'))
      let width = parseFloat(styles.getPropertyValue('width'))
      this.stylus = Object.assign({},this.stylus,{
        width: width
      });
      let time = null
      window.addEventListener('resize', function () {
        clearTimeout(time)
        time = setTimeout(function () {
          if (!that.disabled && that.selected) {
            that.setPosition();
          }
        }, 200)
      })
    },
    beforeDestroy () {
      document.removeEventListener('click', this.backdrop)
      window.removeEventListener('resize', this.backdrop)
      let node = this.$els.dropdownlist
      node && document.body.removeChild(node)
    },
    components: {
      XOption
    },
    events: {
      'select-event': function (newVal) {
        this.value = newVal.value

        this.$dispatch('select-change', newVal)
      }
    }
  }
</script>