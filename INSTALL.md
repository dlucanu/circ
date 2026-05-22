# Version 1.5 with Maude 2.4

1. If you already have [Maude 2.4](https://github.com/maude-lang/Maude/releases/tag/Maude2.4) installed, then go to the next step. Otherwise,
* go to [Maude 2.4 download page](https://github.com/maude-lang/Maude/releases/tag/Maude2.4) download page and follow the steps written there for downloading and installing Maude 2.4;
* if you have VS Code, then you also may use the Maude extension.
1. Download [full-maude24.maude](https://github.com/maude-lang/Maude/releases/download/Maude2.4/full-maude24.maude.gz) and copy it in the folder including Maude tools.
2. Download [circ.maude](./v1.5/circ.maude) fom the folder [v1.5](./v1.5/), which includes the prover.
3. Optionally, download the [examples](./examples/).
4. Follow the instructions from the [manual](./doc/CIRC_Tutorial_v15.pdf), included in [doc/](./doc/) folder, in order to use the Circ tool.

# Version 1.5 with Maude 3.5.1

The steps are the same as above, but you need to download [maude 3.5.1 instead](https://github.com/maude-lang/Maude/releases/tag/Maude3.5.1) of maude 2.4, the file [full-maude351.maude](./v1.5-maude351/full-maude351.maude) instead of full-maude24.maude, and [circ.maude](./v1.5-maude351/circ.maude) from the folder [v1.5-maude351/](./v1.5-maude351/).

The version with Maude 3.5.1 was obtained using Claude. The modifications needed to make CIRC work with Maude 3.5.1 are described in the file [REFACTORING-REPORT.md](./v1.5-maude351/REFACTORING-REPORT.md).

