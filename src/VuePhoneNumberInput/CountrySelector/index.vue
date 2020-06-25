<template>
  <div
    ref="parent"
    :class="[{
      'is-focused': isFocus,
      'has-value': value,
      'has-hint': hint,
      'has-error': error,
      'is-disabled': disabled,
      'no-flags': noFlags,
      'has-list-open': hasListOpen,
      'is-valid': valid
    }]"
    class="position-relative"
    @blur.capture="handleBlur"
    @mouseenter="updateHoverState(true)"
    @mouseleave="updateHoverState(false)"
  >
    <label
      ref="label"
      @click.stop="toggleList"
    >
      {{ hint || label }}
    </label>
    <div class="position-relative ">
      <div
        v-if="value && !noFlags"
        class="position-absolute inset-0 d-flex align-items-center pl-2"
        @click.stop="toggleList"
      >
        <div :class="`iti-flag-small iti-flag ${value.toLowerCase()}`" />
      </div>
      <b-form-input
        :id="id"
        ref="CountrySelector"
        :value="callingCode"
        :placeholder="label"
        :disabled="disabled"
        class="pl-5 user-select-none cursor-pointer"
        readonly
        @focus="isFocus = true"
        @keydown="keyboardNav"
        @click.stop="toggleList"
      />
      <div
        class="position-absolute inset-0 d-flex justify-content-end align-items-center pr-2"
        @click.stop="toggleList"
      >
        <div class="">
          <slot name="arrow" />
        </div>
      </div>
    </div>
    <Transition name="slide">
      <!-- <div
        v-show="hasListOpen"
        ref="countriesList"
        class="dropdown-menu overflow-hidden"
        :class="{ 'has-calling-code': showCodeOnList, 'show': true}"
        :style="[listHeight]"
      > -->
      <div
        v-show="hasListOpen"
        ref="countriesList"
        class="dropdown-menu overflow-hidden overflow-y-auto overflow-x-hidden"
        :class="{ 'has-calling-code': showCodeOnList, 'show': true}"
        :style="[listHeight]"
      >
        <RecycleScroller
          v-slot="{ item }"
          :items="countriesSorted"
          :item-size="1"
          key-field="iso2"
        >
          <button
            :key="`item-${item.code}`"
            :class="[
              { 'selected': value === item.iso2 },
              { 'keyboard-selected': value !== item.iso2 && tmpValue === item.iso2 }
            ]"
            tabindex="-1"
            class="dropdown-item"
            @click.stop="updateValue(item.iso2)"
          >
            <div class="d-flex align-items-center w-100">
              <div
                v-if="!noFlags"
                class="mr-3"
              >
                <div class="w-20px">
                  <div :class="`iti-flag-small iti-flag ${item.iso2.toLowerCase()}`" />
                </div>
              </div>
              <span
                v-if="showCodeOnList"
                class="w-45px text-muted flex-grow-0 flex-shrink-0 flex-basis-auto mr-1"
              >
                +{{ item.dialCode }}
              </span>
              <div class="flex-grow-1">
                {{ item.name }}
              </div>
            </div>
          </button>
        </RecycleScroller>
      </div>
    </Transition>
  </div>
</template>

<script>
  import { getCountryCallingCode } from 'libphonenumber-js'
  import StylesHandler from '@/VuePhoneNumberInput/mixins/StylesHandler'

  import { RecycleScroller } from 'vue-virtual-scroller'

  export default {
    name: 'CountrySelector',
    components: {
      RecycleScroller
    },
    mixins: [StylesHandler],
    props: {
      id: { type: String, default: 'CountrySelector' },
      value: { type: [String, Object], default: null },
      label: { type: String, default: 'Choose country' },
      hint: { type: String, default: String },
      error: { type: Boolean, default: false },
      disabled: { type: Boolean, default: false },
      valid: { type: Boolean, default: false },
      items: { type: Array, default: Array, required: true },
      preferredCountries: { type: Array, default: null },
      onlyCountries: { type: Array, default: null },
      ignoredCountries: { type: Array, default: null },
      noFlags: { type: Boolean, default: false },
      countriesHeight: { type: Number, default: 35 },
      showCodeOnList: { type: Boolean, default: false }
    },
    data () {
      return {
        isFocus: false,
        hasListOpen: false,
        selectedIndex: null,
        tmpValue: this.value,
        query: '',
        indexItemToShow: 0,
        isHover: false
      }
    },
    computed: {
      itemHeight () {
        return {
          height: `${this.countriesHeight}px`
        }
      },
      listHeight () {
        return {
          height: `${(this.countriesHeight + 1) * 7}px`,
          maxHeight: `${(this.countriesHeight + 1) * 7}px`
        }
      },
      countriesList () {
        return this.items.filter(item => !this.ignoredCountries.includes(item.iso2))
      },
      countriesFiltered () {
        const countries = this.onlyCountries || this.preferredCountries
        return countries.map(country => this.countriesList.find(item => item.iso2.includes(country)))
      },
      otherCountries () {
        return this.countriesList.filter(item => !this.preferredCountries.includes(item.iso2))
      },
      countriesSorted () {
        return this.preferredCountries
          ? [ ...this.countriesFiltered,
              ...this.otherCountries ]
          : this.onlyCountries
            ? this.countriesFiltered
            : this.countriesList
      },
      selectedValueIndex () {
        return this.value
          ? this.countriesSorted.findIndex(c => c.iso2 === this.value)
          : null
      },
      tmpValueIndex () {
        return this.countriesSorted.findIndex(c => c.iso2 === this.tmpValue)
      },
      callingCode () {
        return this.value ? `+${getCountryCallingCode(this.value)}` : null
      }
    },
    methods: {
      updateHoverState(value) {
        this.isHover = value
      },
      handleBlur (e) {
        if (this.$el.contains(e.relatedTarget)) return
        this.isFocus = false
        this.closeList()
      },
      toggleList () {
        this.$refs.countriesList.offsetParent ? this.closeList() : this.openList()
      },
      openList () {
        if (!this.disabled) {
          this.$refs.CountrySelector.focus()
          this.$emit('open')
          this.isFocus = true
          this.hasListOpen = true
          if (this.value) this.scrollToSelectedOnFocus(this.selectedValueIndex)
        }
      },
      closeList () {
        this.$emit('close')
        this.hasListOpen = false
      },
      async updateValue (val) {
        this.tmpValue = val
        this.$emit('input', val || null)
        await this.$nextTick()
        this.closeList()
      },
      scrollToSelectedOnFocus (arrayIndex) {
        const countryPositionFromTopInPx = arrayIndex * (this.countriesHeight + 1) - ((this.countriesHeight + 1) * 3)
        this.$nextTick(() => {
          // this.indexItemToShow = arrayIndex - 3
          this.$refs.countriesList.scrollTop = countryPositionFromTopInPx
        })
      },
      keyboardNav (e) {
        const code = e.keyCode
        if (code === 40 || code === 38) {
          // arrow up down
          if (e.view && e.view.event) {
            // TODO : It's not compatible with FireFox
            e.view.event.preventDefault()
          }
          if (!this.hasListOpen) this.openList()
          let index = code === 40 ? this.tmpValueIndex + 1 : this.tmpValueIndex - 1
          if (index === -1 || index >= this.countriesSorted.length) {
            index = index === -1
              ? this.countriesSorted.length - 1
              : 0
          }
          this.tmpValue = this.countriesSorted[index].iso2
          this.scrollToSelectedOnFocus(index)
        } else if (code === 13) {
          // enter
          this.hasListOpen ? this.updateValue(this.tmpValue) : this.openList()
        } else if (code === 27) {
          // escape
          this.closeList()
        } else {
          // typing a country's name
          this.searching(e)
        }
      },
      searching (e) {
        const code = e.keyCode
        clearTimeout(this.queryTimer)
        this.queryTimer = setTimeout(() => {
          this.query = ''
        }, 1000)
        const q = String.fromCharCode(code)
        if (code === 8 && this.query !== '') {
          this.query = this.query.substring(0, this.query.length - 1)
        } else if (/[a-zA-Z-e ]/.test(q)) {
          if (!this.hasListOpen) this.openList()
          this.query += e.key
          const countries = this.preferredCountries ? this.countriesSorted.slice(this.preferredCountries.length) : this.countriesSorted
          const resultIndex = countries.findIndex(c => {
            this.tmpValue = c.iso2
            return c.name.toLowerCase().startsWith(this.query)
          })
          if (resultIndex !== -1) {
            this.scrollToSelectedOnFocus(resultIndex + (this.preferredCountries ? this.preferredCountries.length : 0))
          }
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
  @import '@/assets/scss/variables.scss';
  // @import 'style-helpers';
  @import './assets/iti-flags/flags.css';
  .w-20px {
    width: 20px;
  }

  .w-45px {
    width: 45px;
  }

  .flex-basis-auto {
    flex-basis: auto;
  }

  .cursor-pointer {
    cursor: pointer;
  }

  .overflow-y-auto {
    overflow-y: auto!important;
  }

  .overflow-x-hidden {
      overflow-x: hidden!important;
  }

  .inset-0 {
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
  }
  // Light Theme
  .country-selector {
    z-index: 0;
    user-select: none;

    // &__country-flag {
    //   margin: auto 0;
    //   position: absolute;
    //   top: 21px;
    //   left: 11px;
    //   z-index: 1;
    //   cursor: pointer;

    //   img {
    //     position: absolute;
    //   }
    // }

    &__list {


      &.has-calling-code {
        min-width: 270px;
      }

      &__item {
        padding: 0 10px;
        text-overflow: ellipsis;
        white-space: nowrap;
        overflow: hidden;
        font-size: 12px;
        cursor: pointer;
        background-color: transparent;
        width: 100%;
        border: 0;
        outline: none;

        &__flag-container {
          margin-right: 10px;
        }

        &__calling-code {
          width: 45px;
          color: $muted-color;
        }

        &.hover,
        &.keyboard-selected {
          background-color: $hover-color;
        }

        &.selected {
          color: #FFF;
          font-weight: 600;

          .country-selector__list__item__calling-code {
            color: #FFF;
          }
        }
      }
    }

    &.has-list-open {
      z-index: 1;

      .country-selector {
        &__toggle {
          transform: rotate(180deg);
        }
      }
    }

    &.is-focused {
      z-index: 1;
    }

    &.has-error {
      .country-selector__input {
        border-color: $danger-color;
      }

      .country-selector__label {
        color: $danger-color;
      }
    }

    &.has-value {
      .country-selector__input {
        padding-left: 40px;
      }
    }

    &.has-value,
    &.has-hint {
      .country-selector__label {
        opacity: 1;
        transform: translateY(0);
        font-size: 11px;
      }

      .country-selector__input {
        padding-top: 14px;
      }
    }
  }
</style>
