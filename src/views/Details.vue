<template>
  <div class="container" id="app">
    <div class="d-flex justify-content-center" v-if="loading">
      <div class="spinner-border" role="status">
        <span class="sr-only">Loading...</span>
      </div>
    </div>

    <div v-if="!loading">
      <div class="row">
        <div class="col-12 mb-2 mt-2">
          <span class="float-left">
            <button
              type="button"
              class="btn btn-light btn-lg"
              :disabled="saving"
              v-on:click="goBack()"
            >
              <i class="fad fa-long-arrow-left" />Terug
            </button>
          </span>
          <span class="float-right">
            <button
              type="button"
              class="btn btn-info btn-lg"
              v-on:click="print()"
            >
              <i class="far fa-print" /> Print
            </button>
            &nbsp;
            <button
              type="button"
              class="btn btn-success btn-lg"
              v-on:click="save()"
              :disabled="saving || !hasChanges()"
            >
              <i class="far fa-save" /> Opslaan
            </button>
          </span>
        </div>
      </div>
      <h3>Info</h3>
      <div class="row mb-3">
        <div class="col-4 mb-3">
          <h5>Naam</h5>
          {{klant.voornaam}} {{klant.naam}}
        </div>
        <div class="col-4 mb-3">
          <h5>MVM Nummer</h5>
          {{klant.mvmNummer}}
        </div>
        <div class="col-4 mb-3">
          <h5>Code</h5>
          <span :style="(klant.code || '').indexOf('NB') >= 0 ? 'color: red;' : ''">{{klant.code}}</span>
        </div>
        <div class="col-4 mb-3">
          <h5>Classificatie</h5>
          <span>{{klant.classificatie}}</span>
        </div>
        <div class="col-4">
          <h5>Gezinsleden</h5>
          <pre>{{info.huishouden}}</pre>
        </div>
        <div class="col-4">
          <h5>Opmerking</h5>
          <textarea class="form-control" v-model="opmerking" rows="2"></textarea>
        </div>
      </div>

      <div class="row">
        <div class="col-12 mx-auto">
          <div id="accordion" class="accordion">
            <div v-for="materiaalType in materiaalTypes" v-bind:key="idSafeName(materiaalType)">
              <div v-bind:id="idSafeName(materiaalType)+'heading'" class="card-header bg-white shadow-sm border-0">
                <h6 class="mb-0 font-weight-bold">
                  <a
                    href="#"
                    data-toggle="collapse"
                    v-bind:data-target="'#'+idSafeName(materiaalType)"
                    aria-expanded="false"
                    v-bind:aria-controls="idSafeName(materiaalType)"
                    class="d-block position-relative text-dark text-uppercase collapsible-link py-2"
                  >{{ materiaalType.naam }}</a>
                </h6>
              </div>
              <div
                v-bind:id="idSafeName(materiaalType)"
                v-bind:aria-labelledby="idSafeName(materiaalType)"
                data-parent="#accordion"
                class="collapse"
              >
                <div class="card-body">
                  <div class="row">
                    <h3 class="col-12">
                      <span class="float-right">
                        <button type="button" class="btn btn-success" v-on:click="addRow(materiaalType.ID, materiaalType.naam)">
                          <i class="fas fa-plus-square" /> Rij Toevoegen
                        </button>
                      </span>
                    </h3>
                  </div>
                  <table class="table">
                    <colgroup>
                      <col/>
                      <col/>
                      <col v-if="materiaalType.perKind"/>
                      <col/>
                      <col/>
                      <col v-if="materiaalType.opMaat"/>
                      <col/>
                    </colgroup>
                    <thead class="thead-dark">
                      <tr>
                        <th>Print</th>
                        <th>Datum</th>
                        <th>Aantal</th>
                        <th v-if="materiaalType.perKind">Kind</th>
                        <th>Item</th>
                        <th v-if="materiaalType.opMaat">Maat</th>
                        <th>Opmerking</th>
                        <th></th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr v-for="item in materiaalVoorCategorie(materiaalType.naam)" v-bind:key="item.ID">
                        <td style="width: 50px;">
                          <input type="checkbox" v-model="item.print" />
                        </td>
                        <td style="width: 160px;">
                          <datepicker :format="'yyyy-MM-dd'" v-model="item.datum"></datepicker>
                        </td>
                        <td style="width: 70px;">
                          <input v-model="item.aantal" class="form-control" type="number" min="1" style="width: 60px;"/>
                        </td>
                        <td v-if="materiaalType.perKind">
                          <multiselect v-model="item.ontvanger" :options="getKinderen()" placeholder="Selecteer een"></multiselect>
                        </td>
                        <td>
                          <multiselect v-model="item.object" track-by="naam" label="naam" :options="materiaalType.opties" placeholder="Selecteer een"></multiselect>
                        </td>
                         <td v-if="materiaalType.opMaat">
                          <input
                            v-model="item.maat"
                            class="form-control"
                            placeholder="Maat"
                            style="width: 170px;"
                          />
                        </td>  
                        <td>
                          <input
                            v-model="item.opmerking"
                            class="form-control"
                            placeholder="Opmerking"
                            style="width: 170px;"
                          />
                        </td>
                        <td style="width: 50px;">
                          <button
                            type="button"
                            class="btn btn-outline-danger"
                            v-on:click="removeRow(item)"
                          >
                            <i class="fad fa-trash" />
                          </button>
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { materiaalService } from "../_services/materiaal.service"
import { klantenService } from "../_services/klanten.service"
import * as moment from "moment"

import Datepicker from "vuejs-datepicker";
import Multiselect from "vue-multiselect";

export default {
  props: ["id"],
  components: {
    Datepicker,
    Multiselect
  },
  data: function() {
    return {
      klant: {},
      loading: true,
      originalData: "", //JSON stored here
      saving: false,
      gekregen: [],
      opmerking: "",
      info: {},
      materiaalTypes: {},
      contacten: [],
    };
  },
  methods: {
    validate: function () {
      for (let gekregen of this.gekregen) {
        if (!gekregen.object) {
          throw new Error("item niet ingevuld")
        }
        gekregen.objectId = gekregen.object.ID
        gekregen.datum = moment(new Date(gekregen.datum)).format("YYYY-MM-DDTHH:mm:ssZ") // damn you Go
      }
    },
    idSafeName: function(cat) {
      return cat.naam.replace(" ", "")
    },
    materiaalVoorCategorie: function(catNaam) {
      return this.gekregen.filter(item => item.object.categorie.naam == catNaam)
    },
    getKinderen: function() {
      const out = []
      for (let contact of this.contacten) {
        out.push(`${contact.voornaam} ${contact.naam}`);
      }
      return out;
    },
    addRow: function(catID, catNaam) {
      this.gekregen = [{
        aantal: 1,
        ontvanger: "",
        naam: "",
        maat: "",
        opmerking: "",
        object: {ID: 0, naam: "", categorie: {ID: catID, naam: catNaam}},
        datum: new Date(),
        print: true,
      }].concat(this.gekregen)
    },
    removeRow: function(obj) {
      this.gekregen = this.gekregen.filter(aObj => aObj != obj)
    },
    save: async function () {
      try {
        this.validate()
        const resp = await materiaalService.saveForNumber(this.klant.mvmNummer, { gekregen: this.gekregen, opmerking: this.opmerking })
        this.$Simplert.open({
          title: "Opgeslagen!",
          message: resp.message,
          type: "success",
          customCloseBtnText: "Sluiten"
        });
        this.originalData = JSON.stringify({gekregen: this.gekregen, opmerking: this.opmerking}, getCircularReplacer());
      }catch (e) {
        this.$Simplert.open({
          title: "Error bij opslaan!",
          message: e,
          type: "error",
          customCloseBtnText: "Sluiten"
        });
      }

    },
    hasChanges: function() {
      console.log(this.gekregen)
      if (JSON.stringify({gekregen: this.gekregen, opmerking: this.opmerking}, getCircularReplacer()) != this.originalData) {
        return true;
      }
      return false;
    },
    goBack: function() {
      let vm = this;
      let confirmFn = function() {
        vm.$router.push({ name: "search" });
      };
      if (this.hasChanges()) {
        this.$Simplert.open({
          title: "Er zijn niet opgeslagen wijzigingen!",
          message: "Ben je zeker dat je wil terug gaan?",
          type: "info",
          useConfirmBtn: true,
          onConfirm: confirmFn,
          customConfirmBtnClass: "btn btn-warning",
          customConfirmBtnText: "Ga Terug",
          customCloseBtnText: "Sluiten"
        });
      } else {
        confirmFn();
      }
    }
  },

  created: async function() {
    this.loading = true;

    let materiaalResponse;
    let klantResponse;
    let materiaalOpties;

    try {
      materiaalResponse = await materiaalService.lookUpNumber(this.id);
      klantResponse=  await klantenService.lookUpNumber(this.id);
      this.contacten = await klantenService.getContacten(this.id);
      materiaalOpties = await materiaalService.getObjectOptions();
    }catch (e) {
      this.$Simplert.open({
        title: "Error!",
        message: e,
        type: "error",
        customCloseBtnText: "Sluiten"
      });

      return
    }

    if (materiaalResponse.gekregen) {
      this.gekregen = materiaalResponse.gekregen
    }

    this.originalData = JSON.stringify({gekregen: materiaalResponse.gekregen, opmerking: materiaalResponse.opmerking});

    for (let optie of materiaalOpties) {
      if (!this.materiaalTypes[optie.categorie.naam]) {
        this.materiaalTypes[optie.categorie.naam] = optie.categorie
        this.materiaalTypes[optie.categorie.naam].opties = []
      }
      this.materiaalTypes[optie.categorie.naam].opties.push(optie)
      console.log(this.materiaalTypes[optie.categorie.naam].opties)
    }

    this.klant = klantResponse;
    this.opmerking = materiaalResponse.opmerking
      ? materiaalResponse.opmerking
      : "";

    this.huishoudenData = [];
    for (let contact of this.contacten) {
      // note to future self, you will regret this slice. I told you so!
      this.huishoudenData.push(
        `${contact.geboorteDatum.slice(0,10)} ${contact.geslacht} - ${contact.voornaam} ${contact.naam}`
      );
    }
    this.info.huishouden = this.huishoudenData.sort().join("\n");
    this.loading = false;
  }
};

const getCircularReplacer = () => {
  const seen = new WeakSet();
  return (key, value) => {
    if (typeof value === "object" && value !== null) {
      if (seen.has(value)) {
        return;
      }
      seen.add(value);
    }
    return value;
  };
};
</script>

<style src="vue-multiselect/dist/vue-multiselect.min.css"></style>
<style>
ul {
  padding-left: 20px;
}

/* this is a pain */
.vdp-datepicker {
  width: 150px;
}
.vdp-datepicker input {
  width: 150px;
}
</style>
