# Install zsh

1. Install zsh package

   ```bash
   sudo apt update && apt upgrade -y
   sudo apt install -y zsh curl
   ```

1. Install oh-my-zsh

   ```bash
   sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
   ```

# Install zsh plugins

## zsh-autosuggestions

1. Clone this repository into `$ZSH_CUSTOM/plugins` (by default ~/.oh-my-zsh/custom/plugins)

   ```bash
   git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
   ```

1. Add the plugin to the list of plugins for Oh My Zsh to load (inside `~/.zshrc`)

   ```
   plugins=(
       # other plugins...
       zsh-autosuggestions
   )
   ```

1. Start a new terminal session.

## zsh-syntax-highlighting

1. Clone this repository in oh-my-zsh's plugins directory

   ```bash
   git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
   ```

1. Activate the plugin in ~/.zshrc

   ```
   plugins=( [plugins...] zsh-syntax-highlighting)
   ```

   > [!NOTE]
   > zsh-syntax-highlighting must be the last plugin sourced

1. Restart zsh (such as by opening a new instance of your terminal emulator).
