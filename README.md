# flight-data-project

This GitHub contains all of our project files. The data cleaning is in `airline_data_cleaning.ipynb`, and the data analysis files are in `airline_delay_analysis_modeling.ipynb`. Part of the visualizations are contained in `matplot_visualizations.ipynb` while the other are contained in the JS app with directions below.

---

The visualizations are in an [Observable Framework](https://observablehq.com/framework/) app. To install the required dependencies, run:

```
npm install
```

Then, to start the local preview server, run:

```
npm run dev
```

Then visit <http://localhost:3000> to view the preview page.

For more, see <https://observablehq.com/framework/getting-started>.

## Command reference

| Command           | Description                                              |
| ----------------- | -------------------------------------------------------- |
| `npm install`            | Install or reinstall dependencies                        |
| `npm run dev`        | Start local preview server                               |
| `npm run build`      | Build your static site, generating `./dist`              |
| `npm run deploy`     | Deploy your app to Observable                            |
| `npm run clean`      | Clear the local data loader cache                        |
| `npm run observable` | Run commands like `observable help`                      |
