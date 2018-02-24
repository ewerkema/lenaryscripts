<template>
    <div class="container comparator">
        <div class="comparator-filter">
            <form action="#">
                <div class="filter-group">
                    <h4 class="title">Filteren op</h4>
                </div>
                <hr>
                <div class="filter-group">
                    <label for="comparator-type">Type lening</label>
                    <select name="type" id="comparator-type" v-model="type">
                        <option :value="value" v-for="(text, value) in types">{{ text }}</option>
                    </select>
                </div>
                <hr>
                <div class="filter-group">
                    <label for="comparator-money">Bedrag</label>
                    <input type="text" :value="applyCurrencyFilter(money)" @click="moneyEdit = true" v-if="moneyEdit === false">
                    <input type="number" id="comparator-money-number" v-model="money" @blur="moneyEdit = false" v-else>
                    <input type="range" id="comparator-money" v-model="money" :min="filterOptions.min" :max="filterOptions.max" :step="filterOptions.step" data-rangeSlider>
                    <div class="range-limit">
                        <span class="min">{{ filterOptions.min | currency(0) }}</span>
                        <span class="max">{{ filterOptions.max | currency(0) }}</span>
                    </div>
                </div>
                <hr>
                <div class="filter-group">
                    <label for="comparator-money">Looptijd</label>
                    <div class="input-group">
                        <input type="number" v-model="duration" style="width: 80px;">
                        <input type="text" value="maanden" disabled>
                    </div>
                </div>
            </form>
        </div>

        <div class="comparator-results cols--5">
            <div class="block block-head">
                <div class="cell">Bank</div>
                <div class="cell">Totale aflossing</div>
                <div class="cell">Rente</div>
                <div class="cell">Maandelijkse aflossing</div>
                <div class="cell"></div>
            </div>

            <template v-for="loan in allLoans">
                <a :href="loan.redirect_url" target="_blank" rel="noopener noreferrer" class="block">
                    <div class="cell cell-no-padding-xs">
                    <span v-if="loan.image !== false">
                        <a :href="loan.redirect_url" target="_blank" rel="noopener noreferrer"><img :src="loan.image" /></a>
                    </span>
                        <span v-else>
                        Geen afbeelding gevonden
                    </span>
                    </div>
                    <div class="cell cell-no-padding-xs">
                        <strong class="visible-xs">Totale aflossing</strong>
                        <p>{{ calculateTotalCost(loan.rente_in_jkp) | currency }}</p>
                    </div>
                    <div class="filler">
                        <a href="#" class="btn visible-xs">
                            Ga verder
                        </a>
                    </div>
                    <div class="cell">
                        <strong class="visible-xs">Rente</strong>
                        <p>{{ loan.rente_in_jkp | percentage }}</p>
                    </div>
                    <div class="filler"></div>
                    <div class="cell">
                        <strong class="visible-xs">Maandelijkse aflossing</strong>
                        <p>{{ calculateMonthlyPayment(loan.rente_in_jkp) | currency }}</p>
                    </div>
                    <div class="cell">
                        <a href="#" class="btn hidden-xs">
                            Ga verder
                        </a>
                    </div>

                    <div class="cell cell-full-width">
                        <em>Voorbeeld op basis van Rente: {{ loan.rente_in_jkp | percentage }}, Vaste rente per jaar: {{ loan.rente_in_jkp | percentage }}, Totaal leenbedrag: {{ money | currency }}, Looptijd {{ (duration >= 12) ? Math.ceil(duration/12)+" jaar" : duration+" maand" }}, Totale betaling: {{ calculateTotalCost(loan.rente_in_jkp) | currency }}</em>
                    </div>
                </a>
            </template>
            <div v-if="allLoans.length === 0" class="border">
                <div class="cell cell-full-width" v-if="loading">Bezig met laden van de leningen...</div>
                <div class="cell cell-full-width" v-else>Er zijn geen leningen gevonden voor de huidige selectie. De duurtijd van uw lening is te hoog voor dit bedrag. Geef een lagere duurtijd of hoger bedrag op.</div>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        data() {
            return {
                totalLoanVariations: 20,
                loading: true,
                moneyEdit: false,
                loans: [],
                types: {
                    auto: "Autolening",
                    persoonlijk: "Persoonlijke Lening",
                    renovatie: "Renovatielening",
                    mini: "Minilening",
                    geldreserve: "Geldreserve",
                    groen: "Groene Lening",
                },
                duration: 12,
                type: "persoonlijk",
                money: 5000,
                filterOptions: {},
                defaultFilterOptions: {
                    min: 2500,
                    max: 75000,
                    step: 2500,
                    value: 5000,
                }
            }
        },

        computed: {
            allLoans: function() {
                return this.sortLoans(this.filterLoans(this.loans));
            },
        },

        created() {
            this.getUrlDefaults();
            this.setFilterOptions(this.type);
        },

        mounted() {
            this.getLoans();
        },

        methods: {
            getLoans: function() {
                let self = this;
                let url = (location.hostname === "localhost" || location.hostname === "127.0.0.1") ? 'http://localhost/codecentral/lenarybe/' : '';
                url += '/wp-json/wp/v2/loans/?_embed';
                jQuery.get(url, function (data) {
                    _.each(data, function (loan) {
                        for (let i = 1; i <= self.totalLoanVariations; i++) {
                            if (loan.acf['rente_in_jkp'+i] !== undefined && loan.acf['rente_in_jkp'+i].length > 0) {
                                let frontLoan = {
                                    'image': (loan._embedded !== undefined) ? loan._embedded['wp:featuredmedia'][0].source_url : false,
                                    'redirect_url': loan.acf.redirect_url,
                                    'lening_type': loan.acf.lening_type,
                                    'minimaal_leningbedrag': loan.acf["minimaal_leningbedrag"+i],
                                    'maximaal_leningbedrag': loan.acf["maximaal_leningbedrag"+i],
                                    'minimale_looptijd': loan.acf["minimale_looptijd"+i],
                                    'maximale_looptijd': loan.acf["maximale_looptijd"+i],
                                    'rente_in_jkp': loan.acf["rente_in_jkp"+i],
                                };
                                self.loans.push(frontLoan);
                            }
                        }
                    });
                    self.loading = false;
                });
            },

            getUrlDefaults: function() {
                let type = this.get('type');
                if (type && this.types[type] !== undefined)
                    this.type = type;

                let money = this.get('amount');
                if (money) {
                    this.money = parseInt(money);
                    this.defaultFilterOptions.value = parseInt(money);
                }

                let duration = this.get('duration');
                if (duration)
                    this.duration = parseInt(duration);
            },

            calculateTotalCost: function(interest) {
                return this.calculateMonthlyPayment(interest) * this.duration;
            },

            calculateMonthlyPayment: function(interest) {
                let rate = interest / 1200;
                let months = this.duration;

                return (rate + rate / (Math.pow(1 + rate, months) - 1)) * this.money;
            },

            sortLoans: function(loans) {
                if (loans.length === 0) return [];
                return _.sortBy(loans, [(o) => parseFloat(o.rente_in_jkp)]);
            },

            filterLoans: function(loans) {
                if (loans.length === 0) return [];
                return _.filter(loans, (o) => (
                    parseInt(this.money) > parseInt(o.minimaal_leningbedrag)
                    && parseInt(this.money) <= parseInt(o.maximaal_leningbedrag)
                    && parseInt(this.duration) > parseInt(o.minimale_looptijd)
                    && parseInt(this.duration) <= parseInt(o.maximale_looptijd)
                    && o.lening_type === this.type
                ));
            },

            applyCurrencyFilter(value) {
                return this.$options.filters.currency(value);
            },

            get: function(variable) {
                let query = window.location.search.substring(1);
                let vars = query.split("&");
                for (let i=0; i<vars.length; i++) {
                    let pair = vars[i].split("=");
                    if (pair[0] === variable) return pair[1];
                }
                return false;
            },

            setFilterOptions: function(type) {
                if (type === 'mini')
                    this.setMiniLeningFilterOptions();
                else
                    this.restoreFilterOptions();

                this.updateSlider();
                this.money = this.filterOptions.value;
            },

            setMiniLeningFilterOptions: function() {
                this.filterOptions = {
                    min: 100,
                    max: 2500,
                    step: 50,
                    value: 500,
                };
            },

            restoreFilterOptions: function() {
                this.filterOptions = this.defaultFilterOptions;
            },

            updateSlider: function() {
                let selector = '[data-rangeSlider]',
                    elements = document.querySelectorAll(selector);

                if (elements.length !== 0)
                    elements[0].rangeSlider.update(this.filterOptions);

                return false;
            }
        },

        watch: {
            type: function (value) {
                this.setFilterOptions(value);
            }
        },

        filters: {
            capitalize: function (value) {
                if (!value) return '';
                value = value.toString();
                return value.charAt(0).toUpperCase() + value.slice(1)
            },

            percentage: function (value) {
                if (!value) return '';
                return parseFloat(value).toFixed(2)+'%';
            },

            currency: function (value, decimals = 2) {
                if (!value) return '';
                return parseFloat(value).toLocaleString('nl-NL', {style: 'currency', currency: 'EUR', maximumFractionDigits: decimals, minimumFractionDigits: 0});
            }
        },
    }
</script>
