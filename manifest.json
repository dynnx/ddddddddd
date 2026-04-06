class Provider {
    readonly BASE_URL = "https://anime-sama.to";
    readonly CATALOGUE_URL = "https://anime-sama.to/catalogue/";
    readonly SEANIME_API = "http://127.0.0.1:43211/api/v1/proxy?url=";

    private readonly SUPPORTED_SERVERS = ["sibnet", "vk", "sendvid", "vidmoly", "oneupload"];
    _Server = "";

    getSettings(): Settings {
        return {
            episodeServers: this.SUPPORTED_SERVERS,
            supportsDub: true,
        };
    }

    // 🔎 SEARCH (unchanged, simplified)
    async search(opts: SearchOptions): Promise<SearchResult[]> {
        const searchUrl = `${this.CATALOGUE_URL}?search=${encodeURIComponent(opts.query)}&page=1`;

        const res = await fetch(searchUrl);
        if (!res.ok) return [];

        const html = await res.text();
        const $ = await LoadDoc(html);

        const first = $("#list_catalog > div a").first();
        const url = first.attr("href");

        if (!url) return [];

        return [{
            id: url,
            title: first.text(),
            url: url,
            subOrDub: opts.dub ? "dub" : "sub"
        }];
    }

    // 📺 GET EPISODES (still uses episodes.js but works fine)
    async findEpisodes(id: string): Promise<EpisodeDetails[]> {
        const episodesUrl = `${id}/episodes.js`;

        const res = await fetch(`${this.SEANIME_API}${encodeURIComponent(episodesUrl)}`);
        if (!res.ok) return [];

        const text = await res.text();

        const episodeArrays: string[][] = [];

        for (let i = 0; i < 10; i++) {
            const regex = new RegExp(`var\\s+eps${i}\\s*=\\s*\\[([\\s\\S]*?)\\];`);
            const match = regex.exec(text);

            if (match) {
                const urls = match[1]
                    .split(",")
                    .map(u => u.trim().replace(/['"]/g, ""))
                    .filter(Boolean);

                if (urls.length) episodeArrays.push(urls);
            }
        }

        if (!episodeArrays.length) return [];

        const max = Math.max(...episodeArrays.map(a => a.length));
        const episodes: EpisodeDetails[] = [];

        for (let i = 0; i < max; i++) {
            const urls: string[] = [];

            for (const arr of episodeArrays) {
                if (arr[i]) urls.push(arr[i]);
            }

            if (urls.length) {
                episodes.push({
                    id: urls.join(","),
                    url: id,
                    number: i + 1
                });
            }
        }

        return episodes.reverse();
    }

    // 🎯 SERVER HANDLER (THIS IS THE FIXED PART)
    private async HandleServerUrl(serverUrl: string): Promise<VideoSource[]> {
        try {
            const req = await fetch(`${this.SEANIME_API}${encodeURIComponent(serverUrl)}`);
            if (!req.ok) return [];

            let html = await req.text();

            // ✅ STEP 1 — follow iframe
            const iframeMatch = html.match(/<iframe[^>]+src="([^"]+)"/i);
            if (iframeMatch) {
                let iframeUrl = iframeMatch[1];

                if (!iframeUrl.startsWith("http")) {
                    const base = serverUrl.split("/").slice(0, 3).join("/");
                    iframeUrl = base + iframeUrl;
                }

                const iframeReq = await fetch(`${this.SEANIME_API}${encodeURIComponent(iframeUrl)}`);
                if (iframeReq.ok) {
                    html += await iframeReq.text();
                }
            }

            // ✅ STEP 2 — unpack JS if needed
            function unpack(p: string, a: number, c: number, k: string[]): string {
                while (c--) {
                    if (k[c]) {
                        p = p.replace(new RegExp('\\b' + c.toString(a) + '\\b', 'g'), k[c]);
                    }
                }
                return p;
            }

            let unpacked = "";
            const scripts = html.match(/<script[^>]*>([\s\S]*?)<\/script>/gi) || [];

            for (const s of scripts) {
                if (s.includes("eval(function(p,a,c,k,e,d)")) {
                    const m = s.match(/eval\(function\([^)]*\)\{[\s\S]*?\}\(\s*'([\s\S]*?)'\s*,\s*(\d+)\s*,\s*(\d+)\s*,\s*'([\s\S]*?)'\.split\('\|'\)/);

                    if (m) {
                        unpacked = unpack(m[1], parseInt(m[2]), parseInt(m[3]), m[4].split("|"));
                        break;
                    }
                }
            }

            const combined = html + unpacked;

            // ✅ STEP 3 — extract stream
            const m3u8 = combined.match(/https?:\/\/[^\s'"]+\.m3u8[^\s'"]*/g);
            const mp4 = combined.match(/https?:\/\/[^\s'"]+\.mp4[^\s'"]*/g);

            let urls: string[] = [];

            if (m3u8) {
                urls = m3u8;
            } else if (mp4) {
                urls = mp4;
            }

            urls = [...new Set(urls)];

            return urls.map(url => ({
                url,
                type: url.includes(".m3u8") ? "m3u8" : "mp4",
                quality: `${this._Server} - auto`,
                subtitles: []
            }));

        } catch {
            return [];
        }
    }

    // 🔌 SERVER SELECTOR
    async findEpisodeServer(ep: EpisodeDetails, server: string): Promise<EpisodeServer> {
        this._Server = server;

        const urls = ep.id.split(",");

        const serverUrl = urls.find(u => u.includes(server));

        if (!serverUrl) {
            return {
                headers: {},
                server: "not found",
                videoSources: []
            };
        }

        const sources = await this.HandleServerUrl(serverUrl);

        return {
            headers: { referer: serverUrl },
            server,
            videoSources: sources
        };
    }
}
