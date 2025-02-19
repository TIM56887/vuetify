<template>
  <component :is="tag" class="v-markdown">
    <component :is="template" />
  </component>
</template>

<script setup>
  // Utilities
  import MarkdownIt from 'markdown-it'
  import { compile } from '@vue/compiler-dom'
  import * as vue from 'vue'

  import { VCode } from 'vuetify/components/VCode'
  import { VWindowItem } from 'vuetify/components/VWindow'
  import { VTab } from 'vuetify/components/VTabs'
  import AppMarkup from '@/components/app/Markup.vue'
  import AppFigure from '@/components/app/Figure.vue'
  import AppDivider from '@/components/app/Divider.vue'
  import AppHeading from '@/components/app/Heading.vue'
  import AppLink from '@/components/app/Link.vue'
  import AppTable from '@/components/app/Table.vue'
  import Alert from '@/components/Alert.vue'
  import DocTabs from '@/components/doc/Tabs.vue'

  const md = configureMarkdown(MarkdownIt({
    html: true,
    linkify: true,
    typographer: true,
  }), { headerSections: false })

  md.core.ruler.after('linkify', 'gh_links', state => {
    const blockTokens = state.tokens

    for (let j = 0; j < blockTokens.length; j++) {
      if (blockTokens[j].type !== 'inline') continue

      const tokens = blockTokens[j].children

      for (let i = tokens.length - 1; i >= 0; i--) {
        const currentToken = tokens[i]

        // Skip content of markdown links
        if (currentToken.type === 'link_close') {
          i--
          while (tokens[i].level !== currentToken.level && tokens[i].type !== 'link_open') {
            i--
          }
          continue
        }

        if (currentToken.type === 'text') {
          const text = currentToken.content

          const links = [
            ...text.matchAll(/(?<commits>[a-f0-9]{7,40})(?:\D|$)/gm),
            ...text.matchAll(/(?<issues>#(\d{3,5}))(?:\D|$)/gm),
          ]
          links.sort((a, b) => a.index - b.index)

          const nodes = []
          let level = currentToken.level
          let lastPos = 0

          links.forEach(link => {
            const url = `https://github.com/vuetifyjs/vuetify/${Object.keys(link.groups)[0]}/${link.at(-1)}`
            const linkText = link[1]
            const pos = link.index

            if (pos > lastPos) {
              const token = new state.Token('text', '', 0)
              token.content = text.slice(lastPos, pos)
              token.level = level
              nodes.push(token)
            }

            let token = new state.Token('link_open', 'a', 1)
            token.attrs = [['href', url], ['_target', 'blank']]
            token.level = level++
            nodes.push(token)

            if (link.groups.commits) {
              token = new state.Token('code_inline', 'code', 0)
              token.attrSet('class', 'text-subtitle-1')
            } else {
              token = new state.Token('text', '', 0)
            }
            token.content = linkText
            token.level = level
            nodes.push(token)

            token = new state.Token('link_close', 'a', -1)
            token.level = --level
            nodes.push(token)

            lastPos = link.index + link[1].length
          })
          if (lastPos < text.length) {
            const token = new state.Token('text', '', 0)
            token.content = text.slice(lastPos)
            token.level = level
            nodes.push(token)
          }

          tokens.splice(i, 1, ...nodes)
        }
      }
    }
  })

  const fence = md.renderer.rules.fence
  md.renderer.rules.fence = (tokens, idx, options, env, self) => {
    return fence(tokens, idx, options, env, self)
      .replaceAll('{{', '&lbrace;&lbrace;')
      .replaceAll('}}', '&rbrace;&rbrace;')
  }

  const props = defineProps({
    tag: {
      type: String,
      default: 'div',
    },
    content: String,
  })

  const markdown = computed(() => md.render(props.content, {}))
  const template = computed(() => ({
    // These components are all used in markdown-it-rules
    components: {
      VCode,
      VWindowItem,
      VTab,
      AppMarkup,
      AppFigure,
      AppDivider,
      AppHeading,
      AppLink,
      AppTable,
      Alert,
      DocTabs,
    },
    // eslint-disable-next-line no-new-func
    render: new Function(
      'Vue',
      compile(markdown.value, { hoistStatic: true }).code
    )(vue),
  }))
</script>
