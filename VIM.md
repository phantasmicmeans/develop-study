VIMRC setting 
=================

```bash
" Specify a directory for plugins
" - For Neovim: ~/.local/share/nvim/plugged
" - Avoid using standard Vim directory names like 'plugin'
call plug#begin('~/.vim/plugged')

" Make sure you use single quotes
" Shorthand notation; fetches https://github.com/junegunn/vim-easy-align
Plug 'junegunn/vim-easy-align'

" Any valid git URL is allowed
Plug 'https://github.com/junegunn/vim-github-dashboard.git'

" Multiple Plug commands can be written in a single line using | separators
Plug 'SirVer/ultisnips' | Plug 'honza/vim-snippets'

" On-demand loading
Plug 'scrooloose/nerdtree', { 'on':  'NERDTreeToggle' }
Plug 'tpope/vim-fireplace', { 'for': 'clojure' }

" Using a non-master branch
Plug 'rdnetto/YCM-Generator', { 'branch': 'stable' }

" Using a tagged release; wildcard allowed (requires git 1.9.2 or above)
Plug 'fatih/vim-go', { 'tag': '*' }

" Plugin options
Plug 'nsf/gocode', { 'tag': 'v.20150303', 'rtp': 'vim' }
" Plugin outside ~/.vim/plugged with post-update hook
Plug 'junegunn/fzf', { 'dir': '~/.fzf', 'do': './install --all' }

" Unmanaged plugin (manually installed and updated)
Plug '~/my-prototype-plugin'

Plug 'vim-airline/vim-airline'
Plug 'vim-airline/vim-airline-themes'
Plug 'nathanaelkane/vim-indent-guides'
"Plug 'Vimjas/vim-python-pep8-indent'
Plug 'scrooloose/nerdtree'
Plug 'pangloss/vim-javascript'
"Plug 'mxw/vim-jsx'
"Plug 'Raimondi/delimitMate'
Plug 'Valloric/YouCompleteMe'

" Initialize plugin system
call plug#end()

let g:jsx_ext_required=0
let g:ycm_add_preview_to_completeopt=0
let g:ycm_confirm_extra_conf=0
set completeopt-=preview

set nu ai si hlsearch et sm wmnu
set shiftwidth=4
set tabstop=4
set softtabstop=4
set mouse=a

nnoremap <Tab> >>
nnoremap <S-Tab> <<
inoremap <S-Tab> <C-d>
imap <C-c> <CR><Esc>0
vmap <Tab> >gv
vmap <S-Tab> <gv

set t_Co=256
syntax on

""airline setting
let g:airline_powerline_fonts=1
if !exists('g:airline_symbols')
    let g:airline_symbols={}
endif


let g:airline#extensions#tabline#enabled=1
let g:airline#extensions#tabline#left_sep=' '
let g:airline#extensions#tabline#left_alt_sep='|'
let g:airline_theme='bubblegum'


""NERDTree setting 
map <C-v> :NERDTreeToggle<CR>
map <C-F> :NERDTreeFind<CR>
map <C-A> <C-w>
"nerdtree로 파일 열면 nerdtree창 닫음.
```