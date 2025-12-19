# Floorball Scoreboard

This is a floorball scoreboard that can be used online (https://nonsensation.github.io/floorball-scoreboard/).
It is ment to be used as a Browser Source in your Streaming Software.
It uses the https://saisonmanager.de Api to fetch live data for your floorball match.
This means you need to have a your match statistics on https://saisonmanager.de and enter goals in in real time.

## Alpha

This is a alpha version of the scoreboard.
It is currently being developed and not yet ready for production.


## Features

- 

## Planned Features

- Choose between your own statistics or statistics from saisonmanager.de
- Single framework for data fetching and custom scoreboard overlays for your match

## Developing

Once you've created a project and installed dependencies with `pnpm install`, start a development server:

```sh
pnpm run dev -- --open
```

## Build it yourself

1. This page can be used online (https://nonsensation.github.io/floorball-scoreboard/).

2. You can build it yourself by following the steps below:

```sh
pnpm run build
```

Copy the `index.html` from the `build` folder to your web server or open it locally.

3. You can simply copy the entire webpage from https://nonsensation.github.io/floorball-scoreboard/ into an HTML file and open it locally.
(Right-click into the webpage, select "View page source", copy the entire source code and paste it into an HTML file.)

## License

MIT

## Credits and used Technologies

- Svelte
- SvelteKit
- Vite
- TailwindCSS

