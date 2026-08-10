The individual files are the ones that were modified.  The AcePanel.vue is the important one and the rest get the panel to show up on the Mainsail Dashboard.

AcePanel.vue goes into \src\components\panels
index.ts and types.ts go into \src\store\gui
variables.ts goes into \src\store
Dashboard.vue goes into \src\pages

Once in the correct places you have to use npm to compile it.
Open a cmd prompt and navigate to the root folder of the src files.
Run:

```bash
npm ci
```

this will clean install the dependencies for compiling
Then run:

```bash
npm run build
```

This will create a dist folder inside the mainsail folder.
In the dist folder zip all the files up into whatever you want to name it and unzip in your mainsail folder on your printer.

