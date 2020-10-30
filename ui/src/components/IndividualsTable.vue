<template>
  <v-container>
    <v-row class="actions">
      <h4 class="title">
        <v-icon color="primary" left>
          mdi-account-multiple
        </v-icon>
        Individuals
      </h4>
      <div>
        <v-tooltip bottom transition="expand-y-transition" open-delay="200">
          <template v-slot:activator="{ on }">
            <v-btn
              icon
              v-on="on"
              :disabled="disabledMerge"
              @click="mergeSelected(selectedIndividuals)"
            >
              <v-icon>mdi-call-merge</v-icon>
            </v-btn>
          </template>
          <span>Merge selected</span>
        </v-tooltip>
        <v-tooltip bottom transition="expand-y-transition" open-delay="200">
          <template v-slot:activator="{ on }">
            <v-btn
              icon
              v-on="on"
              :disabled="disabledActions"
              @click="confirmDelete(selectedIndividuals)"
            >
              <v-icon>mdi-delete</v-icon>
            </v-btn>
          </template>
          <span>Delete selected</span>
        </v-tooltip>
      </div>
    </v-row>

    <v-data-table
      hide-default-header
      hide-default-footer
      :headers="headers"
      :items="individuals"
      :expanded.sync="expandedItems"
      item-key="uuid"
      :page.sync="page"
    >
      <template v-slot:item="{ item, expand, isExpanded }">
        <individual-entry
          :name="item.name"
          :organization="item.organization"
          :email="item.email"
          :sources="item.sources"
          :uuid="item.uuid"
          :is-expanded="isExpanded"
          :is-locked="item.isLocked"
          :is-bot="item.isBot"
          :is-selected="item.isSelected"
          :is-highlighted="item.uuid === highlightIndividual"
          @expand="expand(!isExpanded)"
          @saveIndividual="$emit('saveIndividual', item)"
          :draggable="!item.isLocked"
          @dragstart.native="startDrag(item, $event)"
          @dragend.native="removeClass(item, $event)"
          @select="selectIndividual(item)"
          @delete="confirmDelete([item])"
          @merge="mergeSelected([item.uuid, ...$event])"
          @move="move($event)"
          @highlight="$emit('highlight', item)"
          @stopHighlight="$emit('stopHighlight', item)"
        />
      </template>
      <template v-slot:expanded-item="{ item }">
        <expanded-individual
          :uuid="item.uuid"
          :enrollments="item.enrollments"
          :identities="item.identities"
          @unmerge="unmerge($event)"
        />
      </template>
    </v-data-table>

    <div class="text-center pt-2">
      <v-pagination
        v-model="page"
        :length="pageCount"
        :total-visible="7"
        @input="queryIndividuals($event)"
      ></v-pagination>
    </div>

    <v-dialog v-model="dialog.open" max-width="400">
      <v-card>
        <v-card-title class="headline">{{ dialog.title }}</v-card-title>
        <v-card-text>{{ dialog.text }}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn text @click="dialog.open = false">
            Cancel
          </v-btn>
          <v-btn color="blue darken-4" text @click.stop="dialog.action">
            Confirm
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

    <v-card class="dragged-item" color="primary" dark>
      <v-card-title>
        Moving {{ this.selectedIndividuals.length }}
      </v-card-title>
    </v-card>
  </v-container>
</template>

<script>
import {
  mergeIndividuals,
  moveIdentity,
  groupIdentities
} from "../utils/actions";
import IndividualEntry from "./IndividualEntry.vue";
import ExpandedIndividual from "./ExpandedIndividual.vue";

export default {
  name: "IndividualsTable",
  components: {
    IndividualEntry,
    ExpandedIndividual
  },
  props: {
    fetchPage: {
      type: Function,
      required: true
    },
    deleteItem: {
      type: Function,
      required: true
    },
    mergeItems: {
      type: Function,
      required: true
    },
    unmergeItems: {
      type: Function,
      required: true
    },
    itemsPerPage: {
      type: Number,
      required: false,
      default: 10
    },
    highlightIndividual: {
      type: String,
      required: false
    },
    moveItem: {
      type: Function,
      required: true
    }
  },
  data() {
    return {
      headers: [
        { value: "name" },
        { value: "email" },
        { value: "sources" },
        { value: "actions" }
      ],
      individuals: [],
      expandedItems: [],
      pageCount: 0,
      page: 0,
      dialog: {
        open: false,
        title: "",
        text: "",
        action: ""
      }
    };
  },
  computed: {
    disabledActions() {
      return this.selectedIndividuals.length === 0;
    },
    disabledMerge() {
      return this.selectedIndividuals.length < 2;
    },
    selectedIndividuals() {
      return this.individuals.filter(individual => individual.isSelected);
    }
  },
  created() {
    this.queryIndividuals(1);
  },
  methods: {
    async queryIndividuals(page = this.page) {
      let response = await this.fetchPage(page, this.itemsPerPage);
      if (response) {
        this.individuals = this.formatIndividuals(
          response.data.individuals.entities
        );
        this.pageCount = response.data.individuals.pageInfo.numPages;
        this.page = response.data.individuals.pageInfo.page;
        this.$emit("updateIndividuals", this.individuals);
      }
    },
    formatIndividuals(individuals) {
      const formattedList = individuals.map(item => {
        return {
          name: item.profile.name,
          id: item.profile.id,
          email: item.profile.email,
          organization: item.enrollments[0]
            ? item.enrollments[0].organization.name
            : "",
          sources: groupIdentities(item.identities).map(
            identity => identity.name
          ),
          identities: groupIdentities(item.identities),
          enrollments: item.enrollments,
          isLocked: item.isLocked,
          isBot: item.profile.isBot,
          uuid: item.identities[0].uuid,
          isSelected: false
        };
      });

      return formattedList;
    },
    startDrag(item, event) {
      item.isSelected = true;
      event.dataTransfer.dropEffect = "move";
      event.dataTransfer.setData(
        "individuals",
        JSON.stringify(this.selectedIndividuals)
      );
      const dragImage = document.querySelector(".dragged-item");
      event.dataTransfer.setDragImage(dragImage, 0, 0);
    },
    removeClass(item, event) {
      event.target.classList.remove("dragging");
    },
    selectIndividual(individual) {
      individual.isSelected = !individual.isSelected;
    },
    deselectIndividuals() {
      this.individuals.forEach(individual => individual.isSelected = false);
    },
    async deleteIndividuals(individuals) {
      const response = await Promise.all(
        individuals.map(individual => this.deleteItem(individual.uuid))
      );
      if (response) {
        this.queryIndividuals(this.page);
        this.dialog.open = false;
      }
    },
    confirmDelete(individuals) {
      const names = individuals.map(individual => individual.name).toString();
      Object.assign(this.dialog, {
        open: true,
        title: "Delete the selected items?",
        text: names,
        action: () => this.deleteIndividuals(individuals)
      });
    },
    async merge(fromUuids, toUuid) {
      const response = await this.mergeItems(fromUuids, toUuid);
      if (response) {
        this.queryIndividuals(this.page);
        this.dialog.open = false;
        this.$emit("updateWorkspace", {
          update: this.formatIndividuals([response.data.merge.individual]),
          remove: fromUuids
        });
      }
    },
    mergeSelected(individuals) {
      mergeIndividuals(individuals, this.merge, this.dialog);
    },
    async unmerge(uuids) {
      const response = await this.unmergeItems(uuids);
      if (response && response.data) {
        const unmergedItems = this.formatIndividuals(
          response.data.unmergeIdentities.individuals
        );
        this.$emit("saveIndividual", unmergedItems[0]);
        this.queryIndividuals(this.page);
      }
    },
    move(event) {
      moveIdentity(event.fromUuid, event.toUuid, this.moveAction, this.dialog);
    },
    async moveAction(fromUuid, toUuid) {
      this.dialog.open = false;
      const response = await this.moveItem(fromUuid, toUuid);
      if (response) {
        this.queryIndividuals(this.page);
        this.$emit("updateWorkspace", {
          update: this.formatIndividuals([
            response.data.moveIdentity.individual
          ])
        });
      }
    }
  }
};
</script>
<style scoped>
::v-deep .theme--light.v-pagination .v-pagination__item,
::v-deep .theme--light.v-pagination .v-pagination__navigation {
  box-shadow: none;
}
::v-deep button.v-pagination__item {
  transition: none;
}
::v-deep table {
  border-collapse: collapse;
}
::v-deep .v-data-table__wrapper {
  overflow-x: hidden;
}
.actions {
  justify-content: space-between;
  padding: 0 26px 24px 26px;
}
.dragged-item {
  max-width: 300px;
  position: absolute;
  top: -300px;
}
</style>