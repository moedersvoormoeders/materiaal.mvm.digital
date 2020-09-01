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
        <div class="col-4">
          <h5>Huishouden</h5>
          <pre>{{info.huishouden}}</pre>
        </div>
        <div class="col-4">
          <h5>Opmerking</h5>
          <textarea class="form-control" v-model="opmerking" rows="2"></textarea>
        </div>
        <div class="col-4">
          <h5>Totalen</h5>
          <ul>
            <li v-for="total in totals" v-bind:key="total.naam">{{total.naam}}: {{total.aantal}}</li>
          </ul>
        </div>
      </div>

      <div class="row">
        <div class="col-12 mx-auto">
          <div id="accordion" class="accordion">
            <b-card v-for="materiaalType in materiaalTypes" v-bind:key="materiaalType.id">
              <div v-bind:id="materiaalType.id+'heading'" class="card-header bg-white shadow-sm border-0">
                <h6 class="mb-0 font-weight-bold">
                  <a
                    href="#"
                    data-toggle="collapse"
                    v-bind:data-target="'#'+materiaalType.id"
                    aria-expanded="false"
                    v-bind:aria-controls="materiaalType.id"
                    class="d-block position-relative text-dark text-uppercase collapsible-link py-2"
                  >{{ materiaalType.naam }}</a>
                </h6>
              </div>
              <div
                v-bind:id="materiaalType.id"
                v-bind:aria-labelledby="materiaalType.id"
                data-parent="#accordion"
                class="collapse"
              >
                <div class="card-body">
                  <div class="row">
                    <h3 class="col-12">
                      <span class="float-right">
                        <button type="button" class="btn btn-success" v-on:click="addRow(materiaalType.id)">
                          <font-awesome-icon :icon="['fas', 'plus-square']" /> Rij Toevoegen
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
                      <col/>
                    </colgroup>
                    <thead class="thead-dark">
                      <tr>
                        <th>Datum</th>
                        <th>Aantal</th>
                        <th v-if="materiaalType.perKind">Kind</th>
                        <th>Item</th>
                        <th>Opmerking</th>
                        <th></th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr v-for="item in materiaal[materiaalType.id]" v-bind:key="item.id">
                        <td style="width: 160px;">
                          <datepicker :format="'yyyy-MM-dd'" v-model="item.datum"></datepicker>
                        </td>
                        <td style="width: 70px;">
                          <input v-model="item.aantal" class="form-control" type="number" min="1" style="width: 60px;"/>
                        </td>
                        <td v-if="materiaalType.perKind">
                          <multiselect v-model="item.kind" :options="getKinderen()" placeholder="Selecteer een"></multiselect>
                        </td>
                        <td>
                          <multiselect v-model="item.naam" :options="materiaalType.opties" placeholder="Selecteer een"></multiselect>
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
                            v-on:click="removeRow(materiaalType.id, item)"
                          >
                            <font-awesome-icon :icon="['fad', 'trash']" />
                          </button>
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </div>
            </b-card>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { materiaalService } from "../_services/materiaal.service"
import { klantenService } from "../_services/klanten.service"

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
      materiaal: {
        kleding: [],
        speelgoed: [],
        baby: [],
        moeder: [],
      },
      opmerking: "",
      materiaalTypes: {
        kleding: {
          naam: "Kleding",
          id: "kleding",
          perKind: true,
          opties: [
            "Pakket Zomer",
            "Pakket Winter",
            "Schoenen Zomer",
            "Schoenen Winter",
            "Uniform",
          ]
        },
        speelgoed: {
          naam: "Speelgoed",
          id: "speelgoed",
          perKind: true,
          opties: ["Verjaardag", "Mini cadeau", "Carnaval", "Extra"]
        },
        baby: {
          naam: "Baby materiaal",
          id: "baby",
          perKind: false,
          opties: ["die", "hebben", "echt", "veel", "materiaal"]
        },
        moeder: {
          naam: "Voor Moeder",
          id: "moeder",
          perKind: false,
          opties: ["Ziekenhuispakket", "Kindskorf", "Doopsuiker", "Zwangerschapskleding", "Gelegenheids outfit", "Winterjas", "Schoenen", "Kapsalon"]
        }
      }
    };
  },
  computed: {
    totals: function() {
      return [];
    }
  },
  methods: {
    getKinderen: function() {
      const out = []
      // TODO
      //for (let contact of this.contacts) {
      //  out.push(contact.Full_Name);
      //}
      return out;
    },
    addRow: function(catID) {
      this.materiaal[catID]= [{
        aantal: 1,
        kind: "",
        naam: "",
        opmerking: "",
        datum: new Date(),
      }].concat(this.materiaal[catID])
    },
    removeRow: function() {
      // TODO
    },
    hasChanges: function() {
      if (JSON.stringify({gekregen: this.materiaal, opmerking: this.opmerking}) != this.originalData) {
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

    const materiaalResponse = await materiaalService.lookUpNumber(this.id);
    const klantResponse = await klantenService.lookUpNumber(this.id);
    // TODO: kinderen

    this.klant = klantResponse;
    this.info.opmerking = materiaalResponse.opmerking
      ? materiaalResponse.opmerking
      : "";

    this.huishoudenData = [];
    for (let contact of this.contacts) {
      console.log(contact);
      this.huishoudenData.push(
        `${contact.Date_of_Birth} ${contact.Geslacht} - ${contact.Full_Name}`
      );
    }
    this.info.huishouden = this.huishoudenData.sort().join("\n");

    this.loading = false;
  }
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
