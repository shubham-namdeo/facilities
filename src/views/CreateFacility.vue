<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-back-button default-href="/tabs/find-facilities" slot="start" />
        <ion-title>{{ translate("Add Store") }}</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content>
      <main>
        <ion-card>
          <ion-card-header>
            <ion-card-title>{{ translate('Setup Store') }}</ion-card-title>
          </ion-card-header>
          <ion-list>
            <ion-item>
              <ion-select :label="translate('Type')" interface="popover" v-model="selectedFacilityTypeId">
                <ion-select-option :value="facilityTypeId" :key="facilityTypeId" v-for="(type, facilityTypeId) in facilityTypesByParentTypeId">
                  {{ type.description ? type.description : facilityTypeId }}
                </ion-select-option>
              </ion-select>
            </ion-item>
            <ion-item>
              <ion-input label-placement="floating" @ionBlur="setFacilityId($event)" v-model="formData.facilityName">
                <div slot="label">{{ translate('Name') }} <ion-text color="danger">*</ion-text></div>
              </ion-input>
            </ion-item>
            <ion-item lines="none">
              <ion-input :label="translate('Internal ID')" label-placement="floating" ref="facilityId" v-model="formData.facilityId" @ionChange="validateFacilityId" @ionBlur="markFacilityIdTouched" error-text="translate('Internal ID cannot be more than 20 characters.')" />
            </ion-item>
            <ion-item>
              <ion-input :label="translate('External ID')" label-placement="floating" v-model="formData.externalId" />
            </ion-item>
          </ion-list>
        </ion-card>

        <div class="ion-text-center ion-margin">
          <ion-button @click="createFacility()">
            <ion-icon slot="start" :icon="addOutline"/>
            {{ facilityTypes[selectedFacilityTypeId]?.description ? translate(`Create ${facilityTypes[selectedFacilityTypeId].description}`) : translate(`Create ${selectedFacilityTypeId}`) }}
          </ion-button>
        </div>
      </main>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import {
  IonBackButton,
  IonButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonContent,
  IonHeader,
  IonIcon,
  IonInput,
  IonItem,
  IonList,
  IonPage,
  IonSelect,
  IonSelectOption,
  IonText,
  IonTitle,
  IonToolbar,
} from "@ionic/vue";
import { defineComponent } from "vue";
import { mapGetters, useStore } from "vuex";
import { useRouter } from 'vue-router'
import { addOutline } from 'ionicons/icons';
import { translate } from "@hotwax/dxp-components";
import { generateInternalId, showToast } from "@/utils";
import { FacilityService } from "@/services/FacilityService";
import { hasError } from "@/adapter";
import logger from "@/logger";

export default defineComponent({
  name: "CreateFacility",
  components: {
    IonBackButton,
    IonButton,
    IonCard,
    IonCardTitle,
    IonCardHeader,
    IonContent,
    IonHeader,
    IonIcon,
    IonInput,
    IonItem,
    IonList,
    IonPage,
    IonSelect,
    IonSelectOption,
    IonText,
    IonTitle,
    IonToolbar,
  },
  computed: {
    ...mapGetters({
      facilityTypes: "util/getFacilityTypes",
      organizationPartyId: "util/getOrganizationPartyId"
    })
  },
  data() {
    return {
      formData: {
        facilityName: '',
        facilityId: '',
        externalId: '',
      },
      selectedFacilityTypeId: '' as any,
      facilityTypesByParentTypeId: {} as any,
      isAutoGenerateId: true,
    }
  },
  async ionViewWillEnter() {
    this.clearFormData()
    await Promise.all([
      this.store.dispatch('facility/updateCurrentFacility', {}),
      this.store.dispatch('util/fetchFacilityTypes', {
        parentTypeId: 'VIRTUAL_FACILITY',
        parentTypeId_op: 'notEqual',
        facilityTypeId: 'VIRTUAL_FACILITY',
        facilityTypeId_op: 'notEqual'
      })
    ])
    this.facilityTypesByParentTypeId = this.getFacilityTypesByParentTypeId(this.$route.query.type as string)

    // In accordance with the specified requirements, it is essential to treat RETAIL STORE and WAREHOUSE
    // as default elements within the list. These elements may appear at any index within the list structure.
    // Hence to meet requirement we explicitly handling the default nature of RETAIL STORE and WAREHOUSE.
    this.selectedFacilityTypeId = this.facilityTypesByParentTypeId['RETAIL_STORE'] ? 'RETAIL_STORE' : this.facilityTypesByParentTypeId['WAREHOUSE'] ? 'WAREHOUSE' : Object.keys(this.facilityTypesByParentTypeId)[0]
  },
  methods: {
    clearFormData() {
      this.formData = {
        facilityName: '',
        facilityId: '',
        externalId: '',
      }
      this.isAutoGenerateId = true;
    },
    setFacilityId(event: any) {
      if(this.isAutoGenerateId) {
        this.formData.facilityId = generateInternalId(event.target.value)
      }
    },
    async createFacility() {
      if (!this.formData.facilityName?.trim()) {
        showToast(translate('Facility name is required.'))
        return
      }

      if (this.formData.facilityId.length > 20) {
        showToast(translate('Internal ID cannot be more than 20 characters.'))
        return
      }

      // In case the user does not lose focus from the facility name input
      // and click on create the button, we need to set the internal id manually
      if (!this.formData.facilityId) {
        this.formData.facilityId = generateInternalId(this.formData.facilityName)
      }

      try {
        const payload = {
          ...this.formData,
          facilityTypeId: this.selectedFacilityTypeId,
          ownerPartyId: this.organizationPartyId
        }

        const resp = await FacilityService.createFacility(payload);
        if (!hasError(resp)) {
          const { facilityId } = resp.data
          showToast(translate("Facility created successfully."))
          this.store.dispatch('facility/updateCurrentFacility', payload),
          this.router.replace(`/add-facility-address/${facilityId}`)
        } else {
          throw resp.data;
        }
      } catch (error: any) {
        logger.error(error)
        if(error?.response?.data?.error?.message) {
          showToast(error.response.data.error.message)
        } else {
          showToast(translate('Failed to create facility.'))
        }
        return;
      }

      // creating default facility location
      await FacilityService.createFacilityLocation({
        facilityId: this.formData.facilityId,
        locationTypeEnumId: "FLT_PICKLOC",
        areaId: "TL",
        aisleId: "TL",
        sectionId: "TL",
        levelId: "LL",
        positionId: "01",
      })
    },
    getFacilityTypesByParentTypeId(parentTypeId: string) {
      return parentTypeId ? Object.keys(this.facilityTypes).reduce((facilityTypesByParentTypeId: any, facilityTypeId: string) => {
        if (this.facilityTypes[facilityTypeId].parentTypeId === parentTypeId) {
          facilityTypesByParentTypeId[facilityTypeId] = this.facilityTypes[facilityTypeId]
        }
        return facilityTypesByParentTypeId
      }, {}) : this.facilityTypes
    },
    validateFacilityId(event: any) {
      const value = event.target.value;
      (this as any).$refs.facilityId.$el.classList.remove('ion-valid');
      (this as any).$refs.facilityId.$el.classList.remove('ion-invalid');

      if (value === '') return;

      this.formData.facilityId.length <= 20
        ? (this as any).$refs.facilityId.$el.classList.add('ion-valid')
        : (this as any).$refs.facilityId.$el.classList.add('ion-invalid');
      this.isAutoGenerateId = false;
    },
    markFacilityIdTouched() {
      (this as any).$refs.facilityId.$el.classList.add('ion-touched');
    },
  },
  setup() {
    const store = useStore();
    const router = useRouter();

    return {
      addOutline,
      store,
      router,
      translate
    };
  }
});
</script>

<style scoped>
@media (min-width: 700px) {
  main {
    max-width: 375px;
    margin: auto;
  }
}
</style>