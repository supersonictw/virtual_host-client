<template>
  <div :class="activeStatus">
    <Notice v-show="notice" :text="notice" />
    <div v-if="$store.state.loaded" :disabled="latch" class="interactive">
      <v-row>
        <v-col>
          <Profile />
        </v-col>
      </v-row>
      <v-row>
        <v-col>
          <v-btn @click="active = 1">
            <v-icon left>mdi-folder-plus</v-icon>
            New Directory
          </v-btn>
          <v-btn @click="active = 2">
            <v-icon left>mdi-file-plus</v-icon>
            New File
          </v-btn>
        </v-col>
      </v-row>
      <v-row>
        <v-col>
          <v-list v-if="directory.length || cwd.length">
            <v-list-item v-show="cwd.length" @click="enter('..')">
              <v-list-item-icon>
                <v-icon>mdi-reply</v-icon>
              </v-list-item-icon>
              <v-list-item-content>
                <v-list-item-title>..</v-list-item-title>
              </v-list-item-content>
            </v-list-item>
            <v-list-item v-for="(item, index) in directory" :key="index">
              <v-list-item class="mr-2" @click="get(item)">
                <v-list-item-icon>
                  <v-icon v-if="item.type">mdi-folder</v-icon>
                  <v-icon v-else>mdi-file</v-icon>
                </v-list-item-icon>
                <v-list-item-content>
                  <v-list-item-title>{{ item.name }}</v-list-item-title>
                  <v-list-item-subtitle>
                    {{ sizeReadable(item.size) }}
                    {{ timeReadable(item.lastModified) }}
                  </v-list-item-subtitle>
                </v-list-item-content>
              </v-list-item>
              <v-list-item-action>
                <v-menu offset-y>
                  <template #activator="{ on, attrs }">
                    <v-btn title="More" rounded v-bind="attrs" v-on="on">
                      <v-icon>mdi-more</v-icon>
                    </v-btn>
                  </template>
                  <v-list>
                    <v-list-item
                      v-if="item.name.endsWith('.zip')"
                      @click="extract(item.name)"
                    >
                      <v-list-item-icon>
                        <v-icon>mdi-zip-box-outline</v-icon>
                      </v-list-item-icon>
                      <v-list-item-content>Extract</v-list-item-content>
                    </v-list-item>
                    <v-list-item @click="rename(item.name)">
                      <v-list-item-icon>
                        <v-icon>mdi-pen</v-icon>
                      </v-list-item-icon>
                      <v-list-item-content>Rename/Move</v-list-item-content>
                    </v-list-item>
                    <v-list-item @click="compress(item.name)">
                      <v-list-item-icon>
                        <v-icon>mdi-zip-box</v-icon>
                      </v-list-item-icon>
                      <v-list-item-content>Compress</v-list-item-content>
                    </v-list-item>
                    <v-list-item @click="remove(item.name)">
                      <v-list-item-icon>
                        <v-icon>mdi-delete</v-icon>
                      </v-list-item-icon>
                      <v-list-item-content>Remove</v-list-item-content>
                    </v-list-item>
                  </v-list>
                </v-menu>
              </v-list-item-action>
            </v-list-item>
          </v-list>
          <v-list v-else>
            <v-list-item>
              <v-list-item-icon>
                <v-icon>mdi-heart-broken</v-icon>
              </v-list-item-icon>
              <v-list-item-content>
                <v-list-item-title>( Empty )</v-list-item-title>
                <v-list-item-subtitle>
                  There is no file/directory found in your space.
                </v-list-item-subtitle>
              </v-list-item-content>
            </v-list-item>
          </v-list>
        </v-col>
      </v-row>
    </div>
    <div v-else>
      <v-row>
        <v-col class="text-center">
          <v-progress-circular
            indeterminate
            color="amber darken-3"
          ></v-progress-circular>
        </v-col>
      </v-row>
    </div>
    <div class="components">
      <new-directory-model
        v-if="active === 1"
        :cwd="cwd"
        @success="get"
        @cancel="cancel"
      />
      <new-file-model
        v-else-if="active === 2"
        :cwd="cwd"
        @success="get"
        @cancel="cancel"
      />
      <rename-model
        v-else-if="active === 3"
        :cwd="cwd"
        :origin="editing"
        @success="get"
        @cancel="cancel"
      />
      <remove-model
        v-else-if="active === 4"
        :cwd="cwd"
        :target="editing"
        @success="get"
        @cancel="cancel"
      />
      <compress-model
        v-else-if="active === 5"
        :cwd="cwd"
        :origin="editing"
        @success="get"
        @cancel="cancel"
      />
      <extract-model
        v-else-if="active === 6"
        :cwd="cwd"
        :target="editing"
        @success="get"
        @cancel="cancel"
      />
    </div>
  </div>
</template>

<script>
import moment from 'moment'
import filesize from 'filesize'

import Profile from '~/components/user/Profile'
import NewDirectoryModel from '~/components/user/NewDirectoryModel'
import NewFileModel from '~/components/user/NewFileModel'
import RenameModel from '~/components/user/RenameModel'
import RemoveModel from '~/components/user/RemoveModel'
import CompressModel from '~/components/user/CompressModel'
import ExtractModel from '~/components/user/ExtractModel'
import Notice from '~/components/user/Notice'

export default {
  name: 'Dashboard',
  components: {
    Profile,
    NewDirectoryModel,
    NewFileModel,
    RenameModel,
    RemoveModel,
    CompressModel,
    ExtractModel,
    Notice,
  },
  data: () => ({
    cwd: [],
    directory: [],
    editing: '',
    latch: false,
    active: 0,
    notice: '',
  }),
  head: {
    title: 'Dashboard',
  },
  computed: {
    activeStatus() {
      return this.active ? 'active' : ''
    },
  },
  mounted() {
    const accessToken = localStorage.getItem('vhs_auth')
    if (!accessToken) {
      this.$router.push('/')
      return
    }
    this.$axios.setHeader('Authorization', `Bearer ${accessToken}`)
    this.$axios
      .get('profile')
      .then((resp) => {
        this.$store.commit('setProfile', resp.data.data)
        this.$store.commit('setLoaded')
        this.enter()
      })
      .catch(() => {
        localStorage.removeItem('vhs_auth')
        this.notice = 'Authentication failed'
        this.$router.push('/')
      })
  },
  methods: {
    get(item) {
      if (!item) {
        this.cancel()
        this.enter()
        return
      }
      if (item.type) {
        this.enter(item.name)
      } else {
        this.download(item.name)
      }
    },
    async enter(target) {
      if (this.latch) return
      if (target) {
        if (target !== '..') {
          this.cwd.push(target)
        } else {
          this.cwd.pop()
        }
      }
      const targetPath = this.cwd.join('/')
      const result = await this.$axios.$get(`user/${targetPath}`)
      if (!(result.status === 200 && result.data.status)) return
      this.directory = [
        ...result.data.data
          .filter((i) => i.type)
          .sort((a, b) => a.name > b.name),
        ...result.data.data
          .filter((i) => !i.type)
          .sort((a, b) => a.name > b.name),
      ]
    },
    async download(target) {
      if (this.latch) return
      const targetPath = this.cwd.concat(target).join('/')
      const result = await this.$axios.$get(`user/${targetPath}`)
      if (!(result.status === 200 && result.data.status)) return
      const buff = Buffer.from(result.data.data, 'base64')
      const blob = new Blob([buff], { type: 'octet/stream' })
      const link = document.createElement('a')
      link.download = target
      link.href = window.URL.createObjectURL(blob)
      link.click()
      window.URL.revokeObjectURL(link.href)
    },
    rename(target) {
      this.latch = true
      this.editing = target
      this.active = 3
    },
    remove(target) {
      this.latch = true
      this.editing = target
      this.active = 4
    },
    compress(target) {
      this.latch = true
      this.editing = target
      this.active = 5
    },
    extract(target) {
      this.latch = true
      this.editing = target
      this.active = 6
    },
    cancel() {
      this.latch = false
      this.editing = null
      this.active = 0
    },
    sizeReadable(fileSize) {
      return filesize(fileSize)
    },
    timeReadable(nanoTimestamp) {
      const microTimestamp = nanoTimestamp / 1000000
      if (moment.now() - microTimestamp < 86400000) {
        return moment(microTimestamp).fromNow()
      } else {
        return moment(microTimestamp).format()
      }
    },
  },
}
</script>

<style>
.active .interactive {
  opacity: 0.3;
  filter: blur(3px);
}

.active .components {
  position: absolute;
  top: 50px;
  left: 300px;
  right: 300px;
}

@media screen and (max-width: 1200px) {
  .active .components {
    left: 100px;
    right: 100px;
  }
}

@media screen and (max-width: 600px) {
  .active .components {
    left: 50px;
    right: 50px;
  }
}
</style>
