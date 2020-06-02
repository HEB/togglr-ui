<template>
  <div>
    <v-breadcrumbs divider=">" class="pt-0">
      <v-breadcrumbs-item nuxt to="/" exact :disabled="false">Home</v-breadcrumbs-item>
      <v-breadcrumbs-item>Application Details</v-breadcrumbs-item>
    </v-breadcrumbs>
    <div>
      <div class="display-2 mb-4">{{ this.$route.query.appName }}</div>
      <LargeLoadingCard v-if="appDetails.loading" />
      <v-card v-else-if="appDetails.payload">
        <EditFeatureDialog />
        <v-container fluid>
          <span class="title">Application ID</span>
          <v-card flat :color="darkThemeEnabled ? 'darkBackground' : 'lightGrey'" class="mb-4 pa-2">
            <span class="text-xs-left">{{ appDetails.payload.id }}</span>
            <span class="float-right mr-1" @click="copyToClipboard(appDetails.payload.id)">
              <v-icon class="copy-icon">file_copy</v-icon>
            </span>
          </v-card>

          <span class="title">Description</span>
          <v-card
            flat
            :color="darkThemeEnabled ? 'darkBackground' : 'lightGrey'"
            class="mb-4 pa-2"
          >{{ appDetails.payload.descr }}</v-card>

          <v-flex xs12>
            <span class="title">Webhook URL</span>
            <v-text-field
              v-model="webhookUrl"
              data-vv-name="webhookURL"
              data-vv-as="Webhook URL"
              name="webhookURL"
              type="text"
              placeholder="Webhook URL"
              clearable
            ></v-text-field>
            <v-btn
              class="mb-4 ml-0"
              color="primary"
              @click="updateWebhookEvent"
              :loading="updateWebhookObject.loading"
            >Update URL</v-btn>
          </v-flex>

          <div class="buffer"></div>

          <span class="title mt-5">Features</span>
          <v-card flat>
            <v-data-table :headers="featureHeaders" :items="appDetails.payload.featuresById">
              <template slot="items" slot-scope="props">
                <tr @click="props.expanded = !props.expanded">
                  <td>{{ props.item.descr }}</td>
                  <td>
                    <v-switch
                      :ref="'switch-'+props.item.id"
                      :input-value="props.item.active"
                      @change="toggleFeature(props.item)"
                    ></v-switch>
                  </td>
                  <td class="text-xs-center">
                    <v-btn
                      slot="activator"
                      flat
                      color="primary"
                      class="pa-0 ma-0"
                      @click="showEditFeatureDialog({ app: appDetails.payload, feature: props.item })"
                    >
                      <v-icon>edit</v-icon>
                    </v-btn>
                    <v-btn
                      flat
                      color="error"
                      class="pa-0 ma-0"
                      @click="showConfirmCancelDialog(
                      {title:'Delete Feature',
                      description:'You are about to delete the feature ' + props.item.descr  + '. Are you sure?',
                      confirmBtnText:'Yep! Let\'s do this!',
                      cancelBtnText:'Nah, I\'m good.',
                      confirmBtnAction: () => deleteFeatureEvent(props.index)}
                      )"
                    >
                      <v-icon>delete</v-icon>
                    </v-btn>
                  </td>
                </tr>
              </template>
            </v-data-table>
            <v-text-field
              v-validate="'required|specialChars'"
              v-model="featureName"
              required
              :key="featureKey"
              data-vv-name="featureName"
              data-vv-as="Feature Name"
              name="featureName"
              label="Feature Name"
              type="text"
              v-on:keyup.enter="addFeatureEvent"
              clearable
              :error-messages="errors.collect('featureName')"
            ></v-text-field>
            <v-btn
              color="primary"
              class="ml-0"
              :disabled="errors.has('featureName')"
              @click="addFeatureEvent"
              :loading="addInProgress"
            >Add Feature</v-btn>
          </v-card>

          <div class="buffer"></div>

          <span class="title mt-5">Keys</span>
          <v-card flat>
            <v-data-table :headers="keyHeaders" :items="appDetails.payload.keysById">
              <template slot="items" slot-scope="props">
                <tr @click="props.expanded = !props.expanded">
                  <td>{{ props.item.keyName }}</td>
                  <td class="text-xs-center">
                    <v-btn
                      flat
                      color="error"
                      class="pa-0 ma-0"
                      @click="showConfirmCancelDialog(
                    {title:'Delete Feature',
                    description:'You are about to delete the key ' + props.item.keyName  + '. Are you sure?',
                    confirmBtnText:'Yep! Let\'s do this!',
                    cancelBtnText:'Nah, I\'m good.',
                    confirmBtnAction: () => deleteKeyEvent(props.index)}
                    )"
                    >
                      <v-icon>delete</v-icon>
                    </v-btn>
                  </td>
                </tr>
              </template>
            </v-data-table>
            <v-text-field
              v-validate="'required|specialChars'"
              v-model="keyName"
              required
              :key="keyKey"
              data-vv-name="keyName"
              data-vv-as="Key Name"
              name="keyName"
              label="Key Name"
              type="text"
              v-on:keyup.enter="addKeyEvent"
              clearable
              :error-messages="errors.collect('keyName')"
            ></v-text-field>
            <v-btn
              color="primary"
              class="ml-0"
              :disabled="errors.has('keyName')"
              @click="addKeyEvent"
              :loading="addInProgress"
            >Add Key</v-btn>
          </v-card>

          <v-expansion-panel class="elevation-0 ml-0 mt-5">
            <v-expansion-panel-content>
              <div slot="header" class="error--text subheading">Danger zone</div>
              <v-btn
                @click="showConfirmCancelDialog(
                {title:'Delete Application',
                description:'You are about to delete ' + appDetails.payload.name  + '. Are you sure?',
                confirmBtnText:'Yep! Let\'s do this!',
                cancelBtnText:'Nah, I\'m good.',
                confirmBtnAction: () => deleteApplication(appDetails.payload.id)}
                )"
                color="error"
              >Delete Application</v-btn>
            </v-expansion-panel-content>
          </v-expansion-panel>
        </v-container>
      </v-card>
      <v-layout row wrap v-else-if="appDetails.error">
        <v-flex xs12 pa-2 text-xs-center>
          <div class="error--text subheading">{{ appDetails.error }}</div>
          <v-btn
            flat
            color="primary"
            class="pa-0 ma-0"
            @click="retrieveApplicationDetails(this.$route.query.appID)"
          >Retry</v-btn>
        </v-flex>
      </v-layout>
      <div v-else>It's empty</div>
    </div>
  </div>
</template>

<script>
import LargeLoadingCard from "~/components/loading/LargeLoadingCard";
import EditFeatureDialog from "~/components/misc/EditFeatureDialog";
import ConfirmCancelAlert from "~/components/misc/ConfirmCancelAlert";
import { mapActions, mapGetters } from "vuex";
import { Validator } from "vee-validate";
import constants from "~/assets/constants.js";

Validator.extend("specialChars", constants.specialCharValidator);

export default {
  components: {
    LargeLoadingCard,
    EditFeatureDialog,
    ConfirmCancelAlert
  },
  mounted() {
    if (!this.storedApp) {
      this.$router.push("/");
    } else {
      this.webhookUrl = this.storedApp.webhookUrl;
      this.retrieveApplicationDetails(this.storedApp.id);
    }
  },
  data: () => ({
    featureHeaders: [
      { text: "Feature", value: "descr", sortable: false },
      { text: "Enabled", value: "enabled", sortable: false },
      { text: "Actions", align: "center", sortable: false }
    ],
    keyHeaders: [
      { text: "Key", value: "keyName", sortable: false },
      { text: "Actions", align: "center", sortable: false }
    ],
    admins: [],
    dialog: false,
    editFeatureDialog: false,
    loaded: false,
    snackbar: false,
    featureName: "",
    featureKey: 0,
    keyName: "",
    keyKey: 0,
    addInProgress: false,
    webhookUrl: "",
    featuresPagination: {
      descending: true,
      page: 0,
      rowsPerPage: -1,
      sortBy: "descr"
    },
    keysPagination: {
      descending: true,
      page: 0,
      rowsPerPage: -1,
      sortBy: "keyName"
    }
  }),
  computed: {
    darkThemeEnabled() {
      return this.$store.state.settings.darkThemeEnabled;
    },
    storedApp() {
      return this.$store.state.routeStorage.appDetail;
    },
    appDetails() {
      return this.$store.state.applications.appDetails;
    },
    deleteApplicationObject() {
      return this.$store.state.applications.deleteApplication;
    },
    deleteAdminObject() {
      return this.$store.state.applications.deleteAdmin;
    },
    updateWebhookObject() {
      return this.$store.state.applications.updateWebhookObject;
    },
    ...mapGetters({
      getApplicationDetails: "applications/getApplicationDetails"
    })
  },
  methods: {
    changeSort(column) {
      if (this.pagination.sortBy === column) {
        this.pagination.descending = !this.pagination.descending;
      } else {
        this.pagination.sortBy = column;
        this.pagination.descending = false;
      }
    },
    startDeleteApplication(id) {
      this.dialog = false;
      this.deleteApplication(id);
    },
    toggleFeature(feature) {
      this.updateApplication(feature);
    },
    removeAdmin(item) {
      /* Commented out since Admins disabled
      if (this.admins.length <= 1) {
        this.showSnackbar({
          text: "You must have at least one admin for this application"
        });
      } else {
        this.deleteAdmin({
          admin: item,
          url: this.appDetails.payload["_links"].adminsById["href"],
          refreshURL: this.storedApp._links["self"]["href"]
        });
      }
      */
    },
    addFeatureEvent() {
      this.$validator.validate("featureName", this.featureName).then(res => {
        if (res) {
          this.addFeature({
            descr: this.featureName,
            appId: this.appDetails.payload.id,
            active: false,
            id: 0,
            negation: false
          });
          this.featureName = "";
          this.featureKey++;
        }
      });
    },
    deleteFeatureEvent(index) {
      this.deleteFeature(this.appDetails.payload.featuresById[index]);
    },
    addKeyEvent() {
      this.$validator.validate("keyName", this.keyName).then(res => {
        if (res) {
          this.addKey({
            keyName: this.keyName,
            appId: this.appDetails.payload.id
          });
          this.keyName = "";
          this.keyKey++;
        }
      });
    },
    deleteKeyEvent(index) {
      this.deleteKey(this.appDetails.payload.keysById[index]);
    },
    copyToClipboard(textToCopy) {
      this.$copyText(textToCopy);
      this.showSnackbar({
        text: `ID ${textToCopy} copied to clipboard`
      });
    },
    updateWebhookEvent() {
      let newDetails = {
        id: this.appDetails.payload.id,
        webhookUrl: this.webhookUrl
      };

      this.updateWebhook(newDetails);
    },
    ...mapActions({
      retrieveApplicationDetails: "applications/retrieveApplicationDetails",
      deleteApplication: "applications/deleteApplication",
      updateApplication: "applications/updateFeature",
      addAdmin: "applications/addAdmin",
      deleteAdmin: "applications/deleteAdmin",
      addFeature: "applications/addFeature",
      deleteFeature: "applications/deleteFeature",
      addKey: "applications/addKey",
      deleteKey: "applications/deleteKey",
      showSnackbar: "notifications/showSnackbar",
      showEditFeatureDialog: "notifications/showEditFeatureDialog",
      showConfirmCancelDialog: "notifications/showConfirmCancelDialog",
      updateWebhook: "applications/updateWebhook"
    })
  },
  watch: {
    appDetails: {
      handler(object) {
        if (object.payload) {
          this.webhookUrl = object.payload.webhookUrl;
          this.admins = JSON.parse(JSON.stringify(object.payload.adminsById));
        }

        if (object.error) {
          this.showSnackbar({
            text: object.error
          });
        }
      },
      deep: true
    },
    deleteApplicationObject: {
      handler(object) {
        if (object.payload) {
          this.showSnackbar({
            text: "Application deleted successfully"
          });
          this.$router.push("/");
        }
      },
      deep: true
    },
    deleteAdminObject: {
      handler(object) {
        if (object.error) {
          this.showSnackbar({
            text: object.error
          });
        }
      },
      deep: true
    },
    admins: {
      handler(newAdmins, oldAdmins) {
        if (this.loaded) {
          //Adding an admin
          if (newAdmins.length > oldAdmins.length) {
            var adminToAdd = newAdmins.filter(
              admin => !oldAdmins.includes(admin)
            );

            if (adminToAdd.length > 0) {
              this.addAdmin({
                id: adminToAdd[0],
                appId: this.appDetails.payload.id
              });
            }
          }
        }
        this.loaded = true;
      }
    },
    updateWebhookObject: {
      handler(object) {
        if (object.error) {
          this.showSnackbar({
            text: object.error
          });
        } else if (object.payload === "success") {
          this.showSnackbar({
            text: "Webhook URL updated"
          });
        }
      },
      deep: true
    }
  }
};
</script>

<style lang='scss' scoped>
.buffer {
  height: 40px;
}
.copy-icon{
  -webkit-user-select: none; /* Chrome/Safari */        
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* IE10+ */
  -khtml-user-select: none; /* webkit (konqueror) browsers */
  /* Rules below not implemented in browsers yet */
  -o-user-select: none;
  user-select: none;
}
</style>