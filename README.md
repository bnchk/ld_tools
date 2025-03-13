# WMC - Testnet loadtools
<br>
Following are some different loading scripts, beta version run manually not yet fully schedulable.<br>
<br>

* [tx_flooding](./tx_flooding.md) - tx flooding
* [gasburn_mixed](./gasburn_mixed.md) - stress chain with mixed compute/space
* [gasburn_mixed](./gasburn_mixed.md) - stress chain with mixed compute/space
<br>

## Automation Environment Setup steps - quick setup
Quick environment setup command in one step.<br>
Intended for updated Ubuntu 22.04 desktop VM, with snapshot taken in case of issues.<br>
To be run from home folder of standard user (not superuser):<br>

```bash
cd $HOME && \
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash && \
source ~/.bashrc && \
nvm install 20 && \
node -v && \
npm -v && \
mkdir $HOME/wmctest && cd $HOME/wmctest && \
npm init -y && \
npm pkg set type="module" && \
npm install ethers
```
<br>
