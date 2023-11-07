<template>
    <div v-show="loading || !filter.hideWhenEmpty || availableOptions.length > 0" class="pt-2 pb-3">
        <h3 class="px-3 text-xs uppercase font-bold tracking-wide">
            <span>
                {{ filter.name }}
            </span>
        </h3>

        <div class="mt-1 px-3">
            <select-control
                :dusk="`${filter.name}-filter-select`"
                class="w-full block"
                :value="value"
                @change="handleChange"
                :options="availableOptions"
                :label="optionValue"
                :selected="value"
            >
                <option value="" selected>&mdash;</option>
            </select-control>
        </div>
    </div>
</template>

<script>
export default {
    props: {
        resourceName: {
            type: String,
            required: true,
        },
        lens: String,
        filterKey: {
            type: String,
            required: true,
        },
    },

    data: () => ({
        options: [],
        loading: false,
    }),

    created() {
        this.options = this.filter.options
        this.onFilterChanged()
        Nova.$on('filter-changed', this.onFilterChanged)
    },

    methods: {
        onFilterChanged() {
            if (this.loading) {
                return;
            }

            this.loading = true

            const filters = this.filter.dependentOf.reduce((r, filter) => {
                r[filter] = this.$store.getters[`${this.resourceName}/getFilter`](filter).currentValue;
                return r;
            }, {})

            this.fetchOptions(filters)
        },
        handleChange(value) {
            this.$store.commit(`${this.resourceName}/updateFilterState`, {
                filterClass: this.filterKey,
                value: value,
            })

            this.$emit('change')
        },

        optionValue(option) {
            return option.label || option.name || option.value
        },

        async fetchOptions(filters) {
            const lens = this.lens ? `/lens/${this.lens}` : ''
            const {data: options} = await Nova.request().get(`/nova-api/${this.resourceName}${lens}/filters/options`, {
                params: {
                    filters: btoa(JSON.stringify(filters)),
                    filter: this.filterKey,
                },
            })

            this.options = options
            this.loading = false
        }
    },

    computed: {
        filter() {
            return this.$store.getters[`${this.resourceName}/getFilter`](this.filterKey)
        },

        value() {
            return this.filter.currentValue
        },

        availableOptions() {
            let options = _.filter(this.options, option => {
                return !option.hasOwnProperty('depends') || _.every(option.depends, (values, filterName) => {
                    const filter = this.$store.getters[`${this.resourceName}/getFilter`](filterName)
                    if (!filter) return true
                    return _.intersection(
                        _.castArray(filter.currentValue).map(String),
                        _.castArray(values).map(String)
                    ).length > 0;
                })
            })
            if (!this.loading && this.value !== '' && options.filter(option => option.value == this.value).length === 0 ) {
                this.handleChange('')
            }
            return options
        },
    },
}
</script>
