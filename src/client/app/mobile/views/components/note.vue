<template>
<div class="note" :class="{ renote: isRenote }">
	<div class="reply-to" v-if="p.reply && (!os.isSignedIn || os.i.clientSettings.showReplyTarget)">
		<x-sub :note="p.reply"/>
	</div>
	<div class="renote" v-if="isRenote">
		<router-link class="avatar-anchor" :to="note.user | userPage">
			<img class="avatar" :src="`${note.user.avatarUrl}?thumbnail&size=64`" alt="avatar"/>
		</router-link>
		%fa:retweet%
		<span>{{ '%i18n:!@reposted-by%'.substr(0, '%i18n:!@reposted-by%'.indexOf('{')) }}</span>
		<router-link class="name" :to="note.user | userPage">{{ note.user | userName }}</router-link>
		<span>{{ '%i18n:!@reposted-by%'.substr('%i18n:!@reposted-by%'.indexOf('}') + 1) }}</span>
		<mk-time :time="note.createdAt"/>
	</div>
	<article>
		<router-link class="avatar-anchor" :to="p.user | userPage">
			<img class="avatar" :src="`${p.user.avatarUrl}?thumbnail&size=96`" alt="avatar"/>
		</router-link>
		<div class="main">
			<header>
				<router-link class="name" :to="p.user | userPage">{{ p.user | userName }}</router-link>
				<span class="is-bot" v-if="p.user.host === null && p.user.isBot">bot</span>
				<span class="username">@{{ p.user | acct }}</span>
				<div class="info">
					<span class="mobile" v-if="p.viaMobile">%fa:mobile-alt%</span>
					<router-link class="created-at" :to="p | notePage">
						<mk-time :time="p.createdAt"/>
					</router-link>
				</div>
			</header>
			<div class="body">
				<p class="channel" v-if="p.channel != null"><a target="_blank">{{ p.channel.title }}</a>:</p>
				<div class="text">
					<a class="reply" v-if="p.reply">
						%fa:reply%
					</a>
					<mk-note-html v-if="p.text" :text="p.text" :i="os.i" :class="$style.text"/>
					<a class="rp" v-if="p.renote != null">RP:</a>
				</div>
				<div class="media" v-if="p.media.length > 0">
					<mk-media-list :media-list="p.media"/>
				</div>
				<mk-poll v-if="p.poll" :note="p" ref="pollViewer"/>
				<div class="tags" v-if="p.tags && p.tags.length > 0">
					<router-link v-for="tag in p.tags" :key="tag" :to="`/search?q=#${tag}`">{{ tag }}</router-link>
				</div>
				<mk-url-preview v-for="url in urls" :url="url" :key="url"/>
				<a class="location" v-if="p.geo" :href="`http://maps.google.com/maps?q=${p.geo.coordinates[1]},${p.geo.coordinates[0]}`" target="_blank">%fa:map-marker-alt% 位置情報</a>
				<div class="map" v-if="p.geo" ref="map"></div>
				<span class="app" v-if="p.app">via <b>{{ p.app.name }}</b></span>
				<div class="renote" v-if="p.renote">
					<mk-note-preview :note="p.renote"/>
				</div>
			</div>
			<footer>
				<mk-reactions-viewer :note="p" ref="reactionsViewer"/>
				<button @click="reply">
					<template v-if="p.reply">%fa:reply-all%</template>
					<template v-else>%fa:reply%</template>
					<p class="count" v-if="p.repliesCount > 0">{{ p.repliesCount }}</p>
				</button>
				<button @click="renote" title="Renote">
					%fa:retweet%<p class="count" v-if="p.renoteCount > 0">{{ p.renoteCount }}</p>
				</button>
				<button :class="{ reacted: p.myReaction != null }" @click="react" ref="reactButton">
					%fa:plus%<p class="count" v-if="p.reactions_count > 0">{{ p.reactions_count }}</p>
				</button>
				<button class="menu" @click="menu" ref="menuButton">
					%fa:ellipsis-h%
				</button>
			</footer>
		</div>
	</article>
</div>
</template>

<script lang="ts">
import Vue from 'vue';
import parse from '../../../../../text/parse';

import MkNoteMenu from '../../../common/views/components/note-menu.vue';
import MkReactionPicker from '../../../common/views/components/reaction-picker.vue';
import XSub from './note.sub.vue';

export default Vue.extend({
	components: {
		XSub
	},

	props: ['note'],

	data() {
		return {
			connection: null,
			connectionId: null
		};
	},

	computed: {
		isRenote(): boolean {
			return (this.note.renote &&
				this.note.text == null &&
				this.note.mediaIds.length == 0 &&
				this.note.poll == null);
		},
		p(): any {
			return this.isRenote ? this.note.renote : this.note;
		},
		reactionsCount(): number {
			return this.p.reactionCounts
				? Object.keys(this.p.reactionCounts)
					.map(key => this.p.reactionCounts[key])
					.reduce((a, b) => a + b)
				: 0;
		},
		urls(): string[] {
			if (this.p.text) {
				const ast = parse(this.p.text);
				return ast
					.filter(t => (t.type == 'url' || t.type == 'link') && !t.silent)
					.map(t => t.url);
			} else {
				return null;
			}
		}
	},

	created() {
		if ((this as any).os.isSignedIn) {
			this.connection = (this as any).os.stream.getConnection();
			this.connectionId = (this as any).os.stream.use();
		}
	},

	mounted() {
		this.capture(true);

		if ((this as any).os.isSignedIn) {
			this.connection.on('_connected_', this.onStreamConnected);
		}

		// Draw map
		if (this.p.geo) {
			const shouldShowMap = (this as any).os.isSignedIn ? (this as any).os.i.clientSettings.showMaps : true;
			if (shouldShowMap) {
				(this as any).os.getGoogleMaps().then(maps => {
					const uluru = new maps.LatLng(this.p.geo.coordinates[1], this.p.geo.coordinates[0]);
					const map = new maps.Map(this.$refs.map, {
						center: uluru,
						zoom: 15
					});
					new maps.Marker({
						position: uluru,
						map: map
					});
				});
			}
		}
	},

	beforeDestroy() {
		this.decapture(true);

		if ((this as any).os.isSignedIn) {
			this.connection.off('_connected_', this.onStreamConnected);
			(this as any).os.stream.dispose(this.connectionId);
		}
	},

	methods: {
		capture(withHandler = false) {
			if ((this as any).os.isSignedIn) {
				this.connection.send({
					type: 'capture',
					id: this.p.id
				});
				if (withHandler) this.connection.on('note-updated', this.onStreamNoteUpdated);
			}
		},
		decapture(withHandler = false) {
			if ((this as any).os.isSignedIn) {
				this.connection.send({
					type: 'decapture',
					id: this.p.id
				});
				if (withHandler) this.connection.off('note-updated', this.onStreamNoteUpdated);
			}
		},
		onStreamConnected() {
			this.capture();
		},
		onStreamNoteUpdated(data) {
			const note = data.note;
			if (note.id == this.note.id) {
				this.$emit('update:note', note);
			} else if (note.id == this.note.renoteId) {
				this.note.renote = note;
			}
		},
		reply() {
			(this as any).apis.post({
				reply: this.p
			});
		},
		renote() {
			(this as any).apis.post({
				renote: this.p
			});
		},
		react() {
			(this as any).os.new(MkReactionPicker, {
				source: this.$refs.reactButton,
				note: this.p,
				compact: true
			});
		},
		menu() {
			(this as any).os.new(MkNoteMenu, {
				source: this.$refs.menuButton,
				note: this.p,
				compact: true
			});
		}
	}
});
</script>

<style lang="stylus" scoped>
@import '~const.styl'

.note
	font-size 12px
	border-bottom solid 1px #eaeaea

	&:first-child
		border-radius 8px 8px 0 0

		> .renote
			border-radius 8px 8px 0 0

	&:last-of-type
		border-bottom none

	@media (min-width 350px)
		font-size 14px

	@media (min-width 500px)
		font-size 16px

	> .renote
		display flex
		align-items baseline
		padding 8px 16px
		line-height 28px
		color #9dbb00
		background linear-gradient(to bottom, #edfde2 0%, #fff 100%)

		@media (min-width 500px)
			padding 16px

		.avatar-anchor
			display inline-block

			.avatar
				vertical-align bottom
				width 28px
				height 28px
				margin 0 8px 0 0
				border-radius 6px

		[data-fa]
			margin-right 4px

		> span
			flex-shrink 0

			&:last-of-type
				margin-right 8px

		.name
			overflow hidden
			flex-shrink 1
			text-overflow ellipsis
			white-space nowrap
			font-weight bold

		> .mk-time
			display block
			margin-left auto
			flex-shrink 0
			font-size 0.9em

		& + article
			padding-top 8px

	> .reply-to
		background rgba(0, 0, 0, 0.0125)

		> .mk-note-preview
			background transparent

	> article
		padding 14px 16px 9px 16px

		&:after
			content ""
			display block
			clear both

		> .avatar-anchor
			display block
			float left
			margin 0 10px 8px 0
			position -webkit-sticky
			position sticky
			top 62px

			@media (min-width 500px)
				margin-right 16px

			> .avatar
				display block
				width 48px
				height 48px
				margin 0
				border-radius 6px
				vertical-align bottom

				@media (min-width 500px)
					width 58px
					height 58px
					border-radius 8px

		> .main
			float left
			width calc(100% - 58px)

			@media (min-width 500px)
				width calc(100% - 74px)

			> header
				display flex
				align-items center
				white-space nowrap

				@media (min-width 500px)
					margin-bottom 2px

				> .name
					display block
					margin 0 0.5em 0 0
					padding 0
					overflow hidden
					color #627079
					font-size 1em
					font-weight bold
					text-decoration none
					text-overflow ellipsis

					&:hover
						text-decoration underline

				> .is-bot
					margin 0 0.5em 0 0
					padding 1px 6px
					font-size 12px
					color #aaa
					border solid 1px #ddd
					border-radius 3px

				> .username
					margin 0 0.5em 0 0
					overflow hidden
					text-overflow ellipsis
					color #ccc

				> .info
					margin-left auto
					font-size 0.9em

					> .mobile
						margin-right 6px
						color #c0c0c0

					> .created-at
						color #c0c0c0

			> .body

				> .text
					display block
					margin 0
					padding 0
					overflow-wrap break-word
					font-size 1.1em
					color #717171

					>>> .title
						display block
						margin-bottom 4px
						padding 4px
						font-size 90%
						text-align center
						background #eef1f3
						border-radius 4px

					>>> .code
						margin 8px 0

					>>> .quote
						margin 8px
						padding 6px 12px
						color #aaa
						border-left solid 3px #eee

					> .reply
						margin-right 8px
						color #717171

					> .rp
						margin-left 4px
						font-style oblique
						color #a0bf46

					[data-is-me]:after
						content "you"
						padding 0 4px
						margin-left 4px
						font-size 80%
						color $theme-color-foreground
						background $theme-color
						border-radius 4px

				.mk-url-preview
					margin-top 8px

				> .channel
					margin 0

				> .tags
					margin 4px 0 0 0

					> *
						display inline-block
						margin 0 8px 0 0
						padding 2px 8px 2px 16px
						font-size 90%
						color #8d969e
						background #edf0f3
						border-radius 4px

						&:before
							content ""
							display block
							position absolute
							top 0
							bottom 0
							left 4px
							width 8px
							height 8px
							margin auto 0
							background #fff
							border-radius 100%

				> .media
					> img
						display block
						max-width 100%

				> .location
					margin 4px 0
					font-size 12px
					color #ccc

				> .map
					width 100%
					height 200px

					&:empty
						display none

				> .app
					font-size 12px
					color #ccc

				> .mk-poll
					font-size 80%

				> .renote
					margin 8px 0

					> .mk-note-preview
						padding 16px
						border dashed 1px #c0dac6
						border-radius 8px

			> footer
				> button
					margin 0
					padding 8px
					background transparent
					border none
					box-shadow none
					font-size 1em
					color #ddd
					cursor pointer

					&:not(:last-child)
						margin-right 28px

					&:hover
						color #666

					> .count
						display inline
						margin 0 0 0 8px
						color #999

					&.reacted
						color $theme-color

					&.menu
						@media (max-width 350px)
							display none

</style>

<style lang="stylus" module>
.text
	code
		padding 4px 8px
		margin 0 0.5em
		font-size 80%
		color #525252
		background #f8f8f8
		border-radius 2px

	pre > code
		padding 16px
		margin 0
</style>
