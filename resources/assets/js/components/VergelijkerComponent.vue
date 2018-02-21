<template>
    <div class="container">
        <div class="filter">
            <form action="#" class="flex">
                <select name="type" v-model="type">
                    <option :value="value" v-for="(text, value) in types">{{ text }}</option>
                </select>

                <input type="range" v-model="money" min="2500" max="75000" step="2500" data-rangeSlider>
            </form>

            <p>&euro; {{ money }}</p>
            <p>Type lening: {{ type }}</p>
        </div>

        <table>
            <tr>
                <th>Naam</th>
                <th>Bank</th>
                <th>Totale aflossing</th>
                <th>Rente</th>
                <th>Maandelijkse aflossing</th>
            </tr>
            <template v-for="loan in allLoans">
                <tr>
                    <td>{{ loan.title.rendered }}</td>
                    <td><img :src="loan._embedded['wp:featuredmedia'][0].source_url" /></td>
                    <td>{{ calculateTotalCost(loan.acf.rente_in_jkp) | currency }}</td>
                    <td>{{ loan.acf.rente_in_jkp | percentage }}</td>
                    <td>{{ calculateMonthlyPayment(loan.acf.rente_in_jkp) | currency }}</td>
                </tr>
                <tr>
                    <td colspan="5"><em>Voorbeeld op basis van Rente: {{ loan.acf.rente_in_jkp | percentage }}, Vaste rente per jaar: {{ loan.acf.rente_in_jkp | percentage }}, Totaal leenbedrag: {{ money | currency }}, Looptijd {{ duration }} jaar, Totale betaling: {{ calculateTotalCost(loan.acf.rente_in_jkp) | currency }}</em></td>
                </tr>
            </template>
            <tr v-if="allLoans.length === 0">
                <td colspan="5">Er zijn geen leningen gevonden voor de huidige selectie.</td>
            </tr>
        </table>
    </div>
</template>

<style>
    .rangeSlider,.rangeSlider__fill{display:block;-webkit-box-shadow:inset 0 1px 3px rgba(0,0,0,.3);box-shadow:inset 0 1px 3px rgba(0,0,0,.3);border-radius:10px}.rangeSlider{position:relative;background:#7f8c8d}.rangeSlider__horizontal{height:20px;width:100%}.rangeSlider__vertical{height:100%;width:20px}.rangeSlider--disabled{filter:progid:DXImageTransform.Microsoft.Alpha(Opacity=40);opacity:.4}.rangeSlider__fill{background:#16a085;position:absolute;z-index:2}.rangeSlider__fill__horizontal{height:100%;top:0;left:0}.rangeSlider__fill__vertical{width:100%;bottom:0;left:0}.rangeSlider__handle{border:1px solid #ccc;cursor:pointer;display:inline-block;width:40px;height:40px;position:absolute;z-index:3;background:#fff -webkit-gradient(linear,left top,left bottom,from(hsla(0,0%,100%,0)),to(rgba(0,0,0,.1)));background:#fff -o-linear-gradient(hsla(0,0%,100%,0),rgba(0,0,0,.1));background:#fff linear-gradient(hsla(0,0%,100%,0),rgba(0,0,0,.1));-webkit-box-shadow:0 0 8px rgba(0,0,0,.3);box-shadow:0 0 8px rgba(0,0,0,.3);border-radius:50%}.rangeSlider__handle__horizontal{top:-10px}.rangeSlider__handle__vertical{left:-10px;bottom:0}.rangeSlider__handle:after{content:"";display:block;width:18px;height:18px;margin:auto;position:absolute;top:0;right:0;bottom:0;left:0;background-image:-webkit-gradient(linear,left top,left bottom,from(rgba(0,0,0,.13)),to(hsla(0,0%,100%,0)));background-image:-o-linear-gradient(rgba(0,0,0,.13),hsla(0,0%,100%,0));background-image:linear-gradient(rgba(0,0,0,.13),hsla(0,0%,100%,0));border-radius:50%}.rangeSlider__handle:active{background-image:-webkit-gradient(linear,left top,left bottom,from(rgba(0,0,0,.1)),to(rgba(0,0,0,.12)));background-image:-o-linear-gradient(rgba(0,0,0,.1),rgba(0,0,0,.12));background-image:linear-gradient(rgba(0,0,0,.1),rgba(0,0,0,.12))}input[type=range]:focus+.rangeSlider .rangeSlider__handle{-webkit-box-shadow:0 0 8px rgba(142,68,173,.9);box-shadow:0 0 8px rgba(142,68,173,.9)}.rangeSlider__buffer{z-index:1;position:absolute;top:3px;height:14px;background:#2c3e50;border-radius:10px}
    /*# sourceMappingURL=range-slider.css.map*/
    .flex { display: flex; justify-content: space-between; }
    .flex select { margin-right: 50px; }
</style>

<script>
    export default {
        data() {
            return {
                loans: [],
                types: {
                    auto: "Autolening",
                    persoonlijk: "Persoonlijke Lening",
                    renovatie: "Renovatielening",
                    mini: "Minilening",
                    geldreserve: "Geldreserve",
                    groen: "Groene Lening",
                },
                duration: 5,
                type: "auto",
                money: 5000,
            }
        },

        computed: {
            allLoans: function() {
                return this.sortLoans(this.filterLoans(this.loans));
            },
        },

        mounted() {
            this.getLoans();
        },

        methods: {
            getLoans: function() {
                let self = this;
                $.get('http://localhost/codecentral/lenarybe/wp-json/wp/v2/loans/?_embed', function (data) {
                    self.loans = data;
                    console.log(self.loans);
                })
            },

            calculateTotalCost: function(interest) {
                return this.calculateMonthlyPayment(interest) * this.duration * 12;
            },

            calculateMonthlyPayment: function(interest) {
                let rate = interest / 1200;
                let months = this.duration * 12;

                return (rate + rate / (Math.pow(1 + rate, months) - 1)) * this.money;
            },

            sortLoans: function(loans) {
                if (loans.length === 0) return [];
                return _.sortBy(loans, [(o) => o.acf.rente_in_jkp]);
            },

            filterLoans: function(loans) {
                if (loans.length === 0) return [];
                return _.filter(loans, (o) => (
                    parseInt(this.money) >= parseInt(o.acf.minimaal_leningbedrag)
                    && parseInt(o.acf.maximaal_leningbedrag) >= parseInt(this.money)
                    && o.acf.lening_type === this.type
                ));
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

            currency: function (value) {
                if (!value) return '';
                return parseFloat(value).toLocaleString('nl-NL', {style: 'currency', currency: 'EUR'});
            }
        }
    }
</script>
