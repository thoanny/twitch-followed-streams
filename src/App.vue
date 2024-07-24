<script setup>
import { onMounted, ref, onUnmounted } from 'vue';

const clientId = import.meta.env.VITE_TWITCH_CLIENT_ID;
const redirect = import.meta.env.VITE_TWITCH_REDIRECT;
const scope = 'user%3Aread%3Afollows';

// [ ] Remplacer par une clé enregistrée en local
const state = 'gwgXp33t66tkgt5g7WZL8SvFR';

const interval = ref(null);

const loggedIn = ref(false);
const loading = ref(false);

const user = ref({
    login: null,
    user_id: null,
    expires_in: 0,
    token: null,
});

const streams = ref({});
const streamers = ref({});

const clearAll = () => {
    clearInterval(interval.value);
    user.value = {
        login: null,
        user_id: null,
        expires_in: 0,
        token: null,
    };
    loggedIn.value = false;
    streams.value = {};
    localStorage.removeItem('user');
};

onMounted(async () => {
    const searchParams = new URLSearchParams(document.location.hash);

    if (
        searchParams.has('#access_token') &&
        searchParams.has('state') &&
        searchParams.get('state') === state
    ) {
        loading.value = true;
        const token = searchParams.get('#access_token');
        await fetch('https://id.twitch.tv/oauth2/validate', {
            headers: {
                Authorization: `Bearer ${token}`,
            },
        })
            .then((res) => {
                if (!res.ok) {
                    throw new Error(`Response status: ${res.status}`);
                }

                return res.json();
            })
            .then((data) => {
                if (data.client_id !== clientId) {
                    throw new Error(`Bad Client Id`);
                }

                const { login, user_id, expires_in } = data;
                user.value = { login, user_id, expires_in, token };
                localStorage.setItem('user', JSON.stringify(user.value));
                loggedIn.value = true;
            })
            .catch((e) => {
                console.error(e);
            })
            .finally(() => {
                loading.value = false;
            });
        window.history.replaceState({}, document.title, '/');
    }

    const localUser = JSON.parse(localStorage.getItem('user'));
    if (localUser) {
        user.value = localUser;
        loadFollowedStreams();
        interval.value = setInterval(loadFollowedStreams, 1000 * 60 * 5);
        loggedIn.value = true;
    }
});

onUnmounted(() => {
    clearInterval(interval.value);
});

const loadFollowedStreams = async () => {
    loading.value = true;
    await fetch(`https://api.twitch.tv/helix/streams/followed?user_id=${user.value.user_id}`, {
        headers: { Authorization: `Bearer ${user.value.token}`, 'Client-Id': clientId },
    })
        .then((res) => {
            if (!res.ok) {
                throw new Error(`Response status: ${res.status}`);
            }
            return res.json();
        })
        .then(async (d) => {
            document.title = `(${d?.data.length}) Chaînes Twitch suivies en live`;

            const ids = d?.data.map((x) => x.user_id).join('&id=');
            streamers.value = await fetch(`https://api.twitch.tv/helix/users?id=${ids}`, {
                headers: { Authorization: `Bearer ${user.value.token}`, 'Client-Id': clientId },
            }).then((res) => res.json());
            streams.value = d;
        })
        .catch((e) => {
            console.log(e);
            clearAll();
        })
        .finally(() => {
            loading.value = false;
        });

    console.log('loadFollowedStreams');
};

const getAvatar = (uid) => {
    return streamers.value.data.find((s) => s.id == uid)?.profile_image_url;
};
</script>

<template>
    <header class="">
        <div
            class="bg-[--color-twitch] w-full text-white flex gap-4 justify-between items-center p-2 h-[42px]"
        >
            <div class="flex gap-2 items-center">
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="24"
                    height="24"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    class="icon icon-tabler icons-tabler-outline icon-tabler-brand-twitch h-6 w-6"
                >
                    <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                    <path
                        d="M4 5v11a1 1 0 0 0 1 1h2v4l4 -4h5.584c.266 0 .52 -.105 .707 -.293l2.415 -2.414c.187 -.188 .293 -.442 .293 -.708v-8.585a1 1 0 0 0 -1 -1h-14a1 1 0 0 0 -1 1z"
                    />
                    <path d="M16 8l0 4" />
                    <path d="M12 8l0 4" />
                </svg>
                <span class="text-sm font-semibold">
                    En live
                    <span v-if="streams.data">({{ streams.data.length }})</span>
                </span>
            </div>
            <a
                v-if="!loggedIn && !loading"
                :href="`https://id.twitch.tv/oauth2/authorize?response_type=token&client_id=${clientId}&redirect_uri=${redirect}&scope=${scope}&state=${state}`"
                class="btn"
            >
                Connexion
            </a>
            <button @click="clearAll" class="btn" v-if="loggedIn && !loading">Déconnexion</button>
        </div>
    </header>

    <main class="bg-white flex-1">
        <div v-if="loading" class="p-2 text-center text-sm">Chargement en cours...</div>
        <div v-if="!loggedIn && !loading" class="p-2 text-center text-sm">
            Pour voir la liste de vos streameuses et streamers préférés en ligne,
            connectez-vous&nbsp;!
        </div>
        <div v-if="!loading && streams.data" class="flex flex-col gap-2 p-2">
            <a
                v-for="stream in streams.data"
                class="group"
                :href="`https://twitch.tv/${stream.user_login}`"
                target="_blank"
            >
                <div class="flex justify-between gap-4">
                    <div class="flex gap-2 items-center">
                        <img :src="getAvatar(stream.user_id)" alt="" class="h-8 w-8 rounded-full" />
                        <div class="flex flex-col w-full">
                            <div class="font-semibold text-sm leading-4">
                                <span class="group-hover:text-[--color-twitch]">{{
                                    stream.user_name
                                }}</span>
                                {{ stream.mature ? '🔞' : '' }}
                            </div>
                            <div class="text-gray-400 line-clamp-1 text-xs leading-4">
                                {{ stream.game_name }}
                            </div>
                        </div>
                    </div>
                    <div>
                        <div class="whitespace-nowrap text-sm text-gray-600">
                            <span class="text-red-500">●</span>
                            {{
                                stream.viewer_count < 1000
                                    ? stream.viewer_count
                                    : new Intl.NumberFormat().format(
                                          (stream.viewer_count / 1000).toFixed(1),
                                      ) + '&nbsp;k'
                            }}
                        </div>
                    </div>
                </div>
            </a>
        </div>
    </main>
    <footer>
        <div class="text-xs flex gap-2 items-center justify-center p-2 bg-gray-100 text-gray-500">
            <a
                href="https://www.patreon.com/thoanny"
                target="_blank"
                class="flex gap-1 items-center group hover:text-[--color-twitch]"
            >
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="24"
                    height="24"
                    viewBox="0 0 24 24"
                    fill="currentColor"
                    class="icon icon-tabler icons-tabler-filled icon-tabler-brand-patreon w-4 h-4"
                >
                    <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                    <path
                        d="M7.462 3.1c2.615 -1.268 6.226 -1.446 9.063 -.503c2.568 .853 4.471 3.175 4.475 5.81c.004 3.061 -1.942 5.492 -4.896 6.243c-1.693 .43 -2.338 .75 -2.942 1.582c-.238 .328 -.45 .745 -.796 1.533l-.22 .5c-1.146 2.601 -2.156 3.762 -4.236 3.735c-2.232 -.03 -3.603 -1.742 -4.313 -4.48c-.458 -1.768 -.617 -3.808 -.594 -5.876c.044 -3.993 1.42 -7.072 4.46 -8.545z"
                    />
                </svg>
                <span class="group-hover:underline">Patreon</span>
            </a>
            <a
                href="https://www.twitch.tv/subs/thoanny"
                target="_blank"
                class="flex gap-1 items-center group hover:text-[--color-twitch]"
            >
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="24"
                    height="24"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    class="icon icon-tabler icons-tabler-outline icon-tabler-brand-twitch w-4 h-4"
                >
                    <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                    <path
                        d="M4 5v11a1 1 0 0 0 1 1h2v4l4 -4h5.584c.266 0 .52 -.105 .707 -.293l2.415 -2.414c.187 -.188 .293 -.442 .293 -.708v-8.585a1 1 0 0 0 -1 -1h-14a1 1 0 0 0 -1 1z"
                    />
                    <path d="M16 8l0 4" />
                    <path d="M12 8l0 4" />
                </svg>
                <span class="group-hover:underline">Twitch</span>
            </a>
        </div>
    </footer>
</template>
