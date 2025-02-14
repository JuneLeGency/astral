<template>
  <div class="sidebar bg-blue-darkest py-8 overflow-y-scroll px-4">
    <SidebarHeader title="Stars">
      <RefreshButton :active="refreshingStars" @click.native="refreshStars" />
    </SidebarHeader>
    <ul class="dashboard-list sidebar-stars list-none m-0 p-0 pb-3">
      <SidebarItem
        :class="{ selected: noFiltersApplied }"
        :badge="totalStars"
        class="all-stars"
        title="All Stars"
        icon="InboxIcon"
        icon-size="16"
        @click.native="resetFilters"
      />
      <SidebarItem
        :class="{ selected: viewingUntagged }"
        :badge="totalUntaggedStars"
        class="untagged-stars"
        title="Untagged Stars"
        icon="StarIcon"
        icon-size="16"
        @click.native="setViewingUntagged(true)"
      />
    </ul>
    <SidebarGroup :collapsible="true" :is-collapsed="tagsCollapsed" @toggle="toggleCollapsedState('tags')">
      <template v-slot:header="{ toggleCollapsed }">
        <SidebarHeader title="Tags" @click.native.stop="toggleCollapsed">
          <TagSorter />
        </SidebarHeader>
      </template>
      <template v-slot:content="{ isCollapsed }">
        <div v-show="!isCollapsed">
          <NewTagForm @submit="doAddTag" />
          <ul ref="sidebarTags" class="dashboard-list sidebar-tags list-none m-0 p-0 pb-3">
            <SidebarTag
              v-for="tag in tags"
              :key="tag.id"
              :tag="tag"
              :is-selected="currentTag.id == tag.id"
              :data-id="tag.id"
              @click.native="doSetCurrentTag(tag, $event)"
              @starsDropped="tagStarWithData"
              @deleteTag="doDeleteTag"
              @renameTag="doRenameTag"
            />
          </ul>
        </div>
      </template>
    </SidebarGroup>
    <SidebarGroup :collapsible="true" :is-collapsed="languagesCollapsed" @toggle="toggleCollapsedState('languages')">
      <template v-slot:header="{ toggleCollapsed }">
        <SidebarHeader title="Languages" @click.native="toggleCollapsed" />
      </template>
      <template v-slot:content="{ isCollapsed }">
        <ul v-show="!isCollapsed" class="dashboard-list sidebar-languages list-none m-0 p-0 pb-3">
          <SidebarItem
            v-for="lang in languages"
            :key="lang.name"
            :badge="lang.count"
            :title="lang.name"
            :class="{ selected: currentLanguage == lang.name }"
            class="language rounded"
            @click.native="setCurrentLanguage(lang.name)"
          />
        </ul>
      </template>
    </SidebarGroup>
  </div>
</template>
<script>
import NewTagForm from '@/components/Dashboard/Sidebar/NewTagForm'
import RefreshButton from '@/components/Dashboard/Sidebar/RefreshButton'
import SidebarGroup from '@/components/Dashboard/Sidebar/SidebarGroup'
import SidebarHeader from '@/components/Dashboard/Sidebar/SidebarHeader'
import SidebarItem from '@/components/Dashboard/Sidebar/SidebarItem'
import SidebarTag from '@/components/Dashboard/Sidebar/SidebarTag'
import TagSorter from '@/components/Dashboard/Sidebar/TagSorter'
import { mapActions, mapGetters } from 'vuex'
import dragula from 'dragula'
export default {
  name: 'Sidebar',
  components: {
    NewTagForm,
    RefreshButton,
    SidebarGroup,
    SidebarHeader,
    SidebarItem,
    SidebarTag,
    TagSorter
  },
  data() {
    return {
      refreshingStars: false,
      drake: null
    }
  },
  computed: {
    ...mapGetters([
      'tags',
      'currentTag',
      'languages',
      'currentLanguage',
      'viewingUntagged',
      'pageInfo',
      'totalStars',
      'totalUntaggedStars',
      'tagsCollapsed',
      'languagesCollapsed'
    ]),
    noFiltersApplied() {
      return !Object.keys(this.currentTag).length && this.currentLanguage === '' && !this.viewingUntagged
    }
  },
  async mounted() {
    this.$bus.$on('TAGS_SORTED', method => {
      this.sortTags(method)
    })

    this.drake = dragula([this.$refs.sidebarTags]).on('drop', (el, target, source) => {
      let sortMap = Array.from(source.children).map((el, i) => {
        return {
          id: el.dataset.id,
          sort_order: i
        }
      })
      this.reorderTags(sortMap)
    })

    await this.fetchTags()

    if (this.$route.query.tag) {
      const queryTag = this.tags.find(tag => {
        return tag.name === this.$route.query.tag
      })
      if (queryTag) {
        this.setCurrentTag(queryTag)
      }
    }

    if (this.$route.query.language) {
      this.setCurrentLanguage(this.$route.query.language)
    }
  },
  methods: {
    ...mapActions([
      'addTag',
      'fetchTags',
      'setCurrentTag',
      'setCurrentLanguage',
      'setViewingUntagged',
      'addTagToStars',
      'reorderTags',
      'sortTags',
      'deleteTag',
      'renameTag',
      'fetchGitHubStars',
      'cleanupStars',
      'toggleCollapsedState'
    ]),
    async refreshStars() {
      this.refreshingStars = true
      this.$bus.$emit('STATUS', 'Refreshing stars...')
      await this.fetchGitHubStars({ cursor: false, refresh: true })
      while (this.pageInfo.hasNextPage) {
        await this.fetchGitHubStars({
          cursor: this.pageInfo.endCursor,
          refresh: false
        })
      }
      await this.cleanupStars()
      this.refreshingStars = false
      this.$bus.$emit('STATUS', '')
    },
    async doAddTag(name) {
      try {
        await this.addTag(name)
        this.$bus.$emit('NOTIFICATION', `${name} tag added!`)
      } catch (e) {
        this.$bus.$emit('NOTIFICATION', e.errors.name[0], 'error')
      }
    },
    async doDeleteTag({ id, name }) {
      try {
        await this.deleteTag(id)
        this.$bus.$emit('NOTIFICATION', `${name} tag deleted!`)
      } catch {
        this.$bus.$emit('NOTIFICATION', 'Whoops, that tag could not be deleted. Please try again.', 'error')
      }
    },
    async doRenameTag(data) {
      const { id, name } = data
      const oldName = this.tags.find(s => s.id === id).name
      try {
        await this.renameTag(data)
        this.$bus.$emit('NOTIFICATION', `${oldName} tag renamed to ${name}!!`)
      } catch (e) {
        this.$bus.$emit('NOTIFICATION', e.errors.name[0], 'error')
      }
    },
    doSetCurrentTag(tag, e) {
      if (e.target.classList.contains('dashboard-list-item')) {
        this.setCurrentTag(tag)
      }
    },
    resetFilters() {
      this.setViewingUntagged(false)
      this.setCurrentTag({})
      this.setCurrentLanguage('')
    },
    async tagStarWithData({ data, id }) {
      const tag = this.tags.find(tag => tag.id === parseInt(id, 10))
      if (Array.isArray(data)) {
        try {
          await this.addTagToStars({ stars: data, tag })
          this.$bus.$emit('NOTIFICATION', `${tag.name} tag was added to ${data.length} stars!`)
        } catch {
          this.$bus.$emit('NOTIFICATION', 'Whoops, we could not tag that star. Please try again.', 'error')
        }
      } else {
        try {
          await this.addTagToStars({ stars: [data], tag })
          this.$bus.$emit('NOTIFICATION', `${tag.name} tag was added to ${data.nameWithOwner}!`)
        } catch {
          this.$bus.$emit('NOTIFICATION', 'Whoops, we could not tag those stars. Please try again.', 'error')
        }
      }
    }
  }
}
</script>
<style lang="scss">
.gu-mirror {
  position: fixed !important;
  margin: 0 !important;
  z-index: 9999 !important;
  opacity: 0.8;
  -ms-filter: 'progid:DXImageTransform.Microsoft.Alpha(Opacity=80)';
  filter: alpha(opacity=80);
}
.gu-hide {
  display: none !important;
}
.gu-unselectable {
  user-select: none !important;
}
.gu-transit {
  opacity: 0.2;
  -ms-filter: 'progid:DXImageTransform.Microsoft.Alpha(Opacity=20)';
  filter: alpha(opacity=20);
}
</style>
