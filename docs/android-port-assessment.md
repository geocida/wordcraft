# Avaliação do port do WordCraft para Android tablet com S Pen

Estado: **parcial**. As Fases 1 e 2 foram feitas numa sessão na nuvem (Linux x86_64). O protótipo
da Fase 3 está escrito e compila para Android, mas nada rodou no aparelho. A seção do S Pen e o
go/no-go são **provisórios**.

| Campo | Valor |
|---|---|
| Aparelho alvo | Samsung Galaxy Tab S10 FE+ com S Pen |
| Base avaliada | `storytold/wordcraft` `main` em `0a5b4221` (2026-11-02, "Merge pull request #5205"), igual ao `main` de `geocida/wordcraft` |
| Branch deste relatório | `geocida/wordcraft` `android-eval` |
| Protótipo | `geocida/wordcraft-android-probe` `main` em `0528e3b` |
| Toolchain | rustc 1.97.0 (2026-07-07), alvo `aarch64-linux-android` |
| Evidências | `docs/android-eval/logs/*.log` (saídas do cargo) e `docs/android-eval/experimento-android.patch` |

Convenções: caminhos `$CARGO_REGISTRY` e `$CARGO_GIT` nos logs são o cache do cargo
(`~/.cargo/registry/src/index.crates.io-*` e `~/.cargo/git/checkouts`). Toda afirmação traz
evidência (arquivo e linha, ou log). O que não foi verificado diz "não verificado". As horas são
**estimativas de julgamento**, não medidas.

## 1. Resumo e recomendação provisória

**Recomendação provisória: NO-GO para o port nativo agora. GO para testar primeiro o
`wordcraft-web` no Chrome do tablet.** A decisão final depende da Fase 3 no aparelho, em especial
dos itens T2 a T4 (escrever português com o teclado virtual) e P1 a P8 (S Pen) do roteiro do
protótipo.

Por quê:

1. **O núcleo é portável.** `wordcraft-doc`, `-engine`, `-layout`, `-docx`, `-pdf`, `-render` e
   `wordcraft-cli` passam em `cargo check --target aarch64-linux-android` sem nenhuma mudança
   (logs `check-*.log`, todos com "Finished"). É a parte mais cara do produto e não pede nada.
2. **A interface compila com uma mudança pequena.** `wordcraft-ui-egui` falha só porque o
   `craft-update` puxa o `ring` (código C), que exige o NDK (`check-wordcraft-ui-egui.log`). Com o
   `craft-update` fora do grafo do Android, compila (`exp-ui-egui.log`, exit 0).
3. **Os bloqueios estão na camada de plataforma do app desktop.** `rfd` e `arboard` não têm backend
   Android (`exp4-wordcraft.log`). O `eframe` exige escolher NativeActivity ou GameActivity, e o
   `accesskit` (acessibilidade) só funciona com GameActivity (`eframe/src/lib.rs:153` do fork).
   O ponto de entrada `android_main` e a cola JNI exigem `unsafe`, que o workspace proíbe
   (seção 3.5).
4. **O bloqueio que decide é o texto.** No backend Android do winit usado pelo WordCraft, o IME só
   abre e fecha o teclado: `set_ime_cursor_area`, `set_ime_purpose` e `reset_ime` são vazios, e o
   `TextEvent` do GameActivity cai num `warn!("Unknown android_activity input event")`
   (`winit/src/platform_impl/android/mod.rs:916-931` e `:477`). Pela leitura, acentos por toque
   longo, autocorreção e ditado do teclado Samsung não chegariam como composição. Para um
   processador de texto em português, isso é estrutural. **Não verificado no aparelho**
   (roteiro, T2 a T4).
5. **O S Pen chega pela metade.** O winit repassa só posição e pressão (`mod.rs:418`). Hover,
   botões e os demais eventos de caneta viram `None // TODO mouse events` (`mod.rs:388`), e
   `winit::event::Touch` não tem campo para ferramenta nem inclinação (`event.rs:849-865`). A
   tinta do WordCraft usa o ponteiro emulado e grava pontos só com x e y, com uma largura por
   traço (`crates/ui-egui/src/ink.rs:82`, `crates/doc/src/freeform.rs:71-72`,
   `crates/engine/src/cmd/draw.rs:431`).
6. **Manter um fork é caro.** O upstream integrou 368 PRs e 1.367 commits de 2026-10-19 a
   2026-11-02, com pico de 194 merges em 2026-10-22, e mexeu em 245 arquivos de `apps/wordcraft`,
   `crates/ui-egui` e `Cargo.toml` no mesmo período (seção 8).
7. **A web já resolve parte do problema.** O fork do egui tem trabalho grande de composição de
   texto na web (`crates/eframe/src/web/text_agent.rs`, +1.349 linhas sobre o egui 0.37.0). O
   `wordcraft-web` existe e roda com `trunk serve` (`README.md:167`). Também não usa pressão
   (seção 9). **Não verificado no aparelho**.

Estimativa de esforço do port nativo (julgamento; detalhes na seção 6):

| Fase | O que entrega | Horas |
|---|---|---|
| A | Abre, renderiza e rola um documento no tablet | 24 a 40 |
| B | Usável para escrever: IME com composição, abrir e salvar pelo SAF, clipboard, ciclo de vida | 100 a 180 |
| C | S Pen de verdade: hover, botões, ferramenta, inclinação, pressão na tinta | 40 a 80 |
| D | Uso confortável: interface adaptada ao toque, impressão, leitura em voz alta, acessibilidade | não estimado (depende de decisões de produto) |

O que mudaria a recomendação para GO:

1. O teste T2 mostrar que o teclado Samsung entrega acentos e autocorreção como `KeyEvent`
   utilizáveis mesmo sem composição (improvável pela leitura, mas é o teste que decide).
2. Os mantenedores do `storytold/winit` aceitarem um patch de IME e caneta para Android, o que
   transforma o custo de fork em contribuição.
3. A web no Chrome falhar em algo essencial que o nativo resolva (por exemplo, abrir e salvar
   arquivos locais de forma prática).

## 2. Método e limites desta sessão

1. **Sem NDK.** O download do NDK foi negado pela política de rede da sessão
   (`dl.google.com:443`, resposta 403 do proxy). Por isso a Fase 2 usou `cargo check` (análise de
   tipos, sem ligação). Código C e C++ não foi compilado.
2. **Tentativa de contornar só para o C:** um wrapper com o clang 18 do sistema
   (`--target=aarch64-linux-android26 -ffreestanding`) falhou por falta dos headers do Bionic
   (`check3-wordcraft-ui-egui.log`: "fatal error: 'bits/wordsize.h' file not found"). Não insisti.
3. **Experimento fora do branch.** Para ver o que quebra depois do `ring`, fiz mudanças numa cópia
   separada (git worktree). Elas não estão aplicadas no branch; estão em
   `docs/android-eval/experimento-android.patch` para reproduzir.
4. **Sem aparelho.** Nada foi instalado nem testado no tablet. A Fase 3 ficou com o protótipo e o
   roteiro.
5. **Ambiente diferente do pedido.** O pedido era WSL; a sessão rodou num contêiner Linux. Para
   `cargo check` a diferença não importa; para adb e Gradle, sim.

## 3. Fase 1: inventário estático

### 3.1 Dependências por alvo que mudam no Android

| Arquivo:linha | cfg | Dependência | No Android |
|---|---|---|---|
| `apps/wordcraft/Cargo.toml:22` | todos | `eframe` com `accesskit`, `wayland`, `x11`, `clipboard`, `wgpu` | falta um backend Android; `accesskit` exige GameActivity (`exp2-wordcraft.log`) |
| `apps/wordcraft/Cargo.toml:52-60` | `all(unix, not(macos))` | `winit` com `x11`, `zbus`, `wl-clipboard-rs` | entra no Android. Compilam (`dep-*.log`), mas não servem: o winit não monta X11 no Android (build.rs do winit, linhas 10, 16 e 20); não há D-Bus nem Wayland |
| `apps/wordcraft/Cargo.toml:75-85` | `not(wasm32)` | `craft-print`, `rfd`, `pollster`, `arboard`, `getrandom` | `rfd` e `arboard` não compilam (`exp4-wordcraft.log`); `craft-print` compila e não imprime (3.3) |
| `crates/ui-egui/Cargo.toml:41-42` | `target_os = "linux"` | `zbus` | não entra no Android (`linux` não casa com `android`) |
| `crates/ui-egui/Cargo.toml:45-47` | `not(wasm32)` | `chrono` com `clock`, `craft-update` | `craft-update` puxa `ureq`, `rustls` e `ring` (`cargo tree -i ring`); `ring` exige o NDK |
| `crates/ui-egui/Cargo.toml:36` | todos | `sys-locale` | compila e tem backend Android (`sys-locale/src/android.rs`) |
| `crates/engine/Cargo.toml:42-46` | `not(wasm32)`, `unix` | `getrandom`, `libc` | compilam (`check-engine.log`) |
| `crates/render/Cargo.toml:21-22` | `not(wasm32)` | `vello_cpu` com `multithreading` | compila (`check-render.log`) |

### 3.2 Blocos cfg no código que capturam o Android

O padrão `all(unix, not(target_os = "macos"))` aparece 14 vezes e trata o Android como
Linux/BSD de desktop. Os que importam:

| Arquivo:linha | O que faz | No Android |
|---|---|---|
| `apps/wordcraft/src/main.rs:462-468` | força X11 via `winit::platform::x11` | não compila: o módulo `x11` do winit não existe no Android (`winit/src/platform/mod.rs:21-22`). Inferido pela leitura; o compilador parou antes, no `rfd` |
| `apps/wordcraft/src/file_dialogs.rs:64-75, 113-200` | portal XDG, zenity, `dbus-send` | sem sentido no Android; precisa do SAF |
| `apps/wordcraft/src/default_apps.rs:26` | `xdg::make_default` | sem sentido no Android |
| `apps/wordcraft/src/animations.rs:24` e `animations/desktop.rs:109-112` | lê configurações de animação do KDE/GTK e observa arquivos | sem sentido no Android |
| `apps/wordcraft/src/clipboard.rs:142-145` | Wayland data-control | já exclui o Android |
| `crates/engine/src/speech.rs:877` | leitura em voz alta por `espeak-ng` | não há `espeak-ng` no Android; precisa do TextToSpeech por JNI |
| `crates/ui-egui/src/backstage.rs:629` | botão "Browse in WordCraft…" | aparece no Android (inofensivo, talvez útil) |
| `crates/engine/src/io/safe_save.rs:48, 161, 315-435` | gravação segura com chamadas unix | compila; funciona em caminhos de arquivo, não em URIs do SAF. Não verificado em execução |
| `crates/zotero/src/peer.rs:40, 465` | já trata `android` junto com `linux` | compila |

### 3.3 Crates de plataforma: funciona, tem substituto ou precisa de código novo

| Crate | Situação no Android | Evidência | Veredito |
|---|---|---|---|
| `rfd` 0.17.2 | sem backend Android (backends: gtk3, linux, macos, wasm, win_cid, xdg_desktop_portal) | `exp4-wordcraft.log`: "the trait bound `FileDialog: FilePickerDialogImpl` is not satisfied" em `rfd-0.17.2/src/file_dialog.rs:123` | **código novo**: SAF (`ACTION_OPEN_DOCUMENT`, `ACTION_CREATE_DOCUMENT`) por JNI. O engine já abre e salva a partir de bytes (`crates/engine/src/io.rs:146`, `:417`) |
| `arboard` 3.7.0 | exclui o Android explicitamente | `arboard-3.7.0/src/platform/mod.rs:1-5`; `exp4-wordcraft.log`: "cannot find `Clipboard` in `platform`" em `arboard-3.7.0/src/lib.rs:82` | **código novo**: `ClipboardManager` por JNI. O egui-winit também tira o `arboard` no Android (`egui-winit/Cargo.toml:135-136` do fork), então copiar e colar do próprio egui fica sem a área de transferência do sistema. Não verificado em execução |
| `craft-print` (craft-libs `92d392c`) | compila; no unix usa `lpstat`, `lpoptions` e `lp` do CUPS | `dep-craft-print.log` (exit 0); `craft-libs/crates/print/src/lib.rs:299-330, 395-400, 516-530` | **código novo**: no Android não há CUPS; a lista de impressoras volta vazia e o envio dá erro "the print spooler is not available" (lib.rs:526). Substituto: `PrintManager` por JNI, ou exportar PDF e compartilhar |
| `craft-window` (craft-libs `92d392c`) | compila | `dep-craft-window.log` (exit 0) | **não verificado em execução**: uma janela nativa por documento não existe no Android; o comportamento de viewports extras do eframe no Android não foi testado |
| `craft-update` (craft-libs `c5a1cbf`) | puxa `ureq` com `rustls` e `ring` | `craft-libs/crates/update/Cargo.toml:30`; `check-wordcraft-ui-egui.log` | **retirar no Android** (cfg). Atualização vem da loja ou do APK; não faz sentido um atualizador próprio |
| `sys-locale` 0.3 | backend Android lê `persist.sys.locale` | `sys-locale/src/android.rs:20-33`; `dep-sys-locale.log` | **funciona** (compila; não verificado em execução) |
| Hunspell (`craft-proof`, craft-libs `6e45fda`) | procura dicionários em `/usr/share/hunspell` e afins | `craft-libs/crates/proof/src/hunspell.rs:125-128`; `dep-craft-proof.log` | **funciona com ajuste**: as pastas não existem no Android; a saída é embutir os dados do craft-i18n no build (`CRAFT_I18N_DIR`, ver `CLAUDE.md`, seção Assets) |
| Fontes do sistema | `system_font_dirs` cai no ramo Linux | `crates/fonts/src/fontdb/names.rs:230-238` | **ajuste pequeno**: acrescentar `/system/fonts` (caminho padrão do Android; não verificado no aparelho) |
| `accesskit` | só com GameActivity | `eframe/src/lib.rs:153` do fork: "`accesskit` feature is only available with `android-game-activity`" | **depende da escolha de atividade** (3.5) |
| `zbus`, `wl-clipboard-rs` | compilam | `dep-zbus@5.19.0.log`, `dep-wl-clipboard-rs.log` | **retirar no Android** (cfg): sem D-Bus e sem Wayland |

### 3.4 Forks storytold/egui e storytold/winit contra o upstream

| Fork | Revisão usada | Base upstream | Distância | Backend Android |
|---|---|---|---|---|
| `storytold/winit` | `a517b384` (2026-10-24) | `v0.30.13` de `rust-windowing/winit` (ancestral confirmado com `git merge-base --is-ancestor`) | 37 commits, 64 arquivos, +20.456 linhas; quase tudo Wayland, X11, macOS e Windows (lista W1 a W27 no `PATCHES.md` do fork) | **presente e compila** (`dep-winit.log`). Única mudança Android: `reset_ime` e `reset_ime_with_request` vazios (`mod.rs:928-931`); `Cargo.toml` mantém `android-native-activity` e `android-game-activity` (linhas 76-77) |
| `storytold/egui` | `6d518fc7` | `0.37.0` de `emilk/egui` (ancestral confirmado) | 85 commits, 108 arquivos, +34.363 linhas; foco em IME, clipboard e web | **presente e compila** (`dep-eframe.log`, `dep-egui-wgpu.log`). O `eframe` e o `egui-winit` mantêm `android-native-activity` e `android-game-activity` (`eframe/Cargo.toml:50-52`, `egui-winit/Cargo.toml:37-39`); o `NativeOptions.android_app` segue lá (`eframe/src/epi.rs:433`) |

O `egui_glow` isolado não pôde ser checado: o cargo entrou em pânico ao resolver features com
`-p egui_glow` (`dep-egui_glow.log`, "panicked at .../resolver/features.rs:325"). O protótipo
compila com `glow` ligado, o que cobre o mesmo código por outro caminho (seção 5).

Observação: o upstream do winit já tem a série 0.31 (tags `v0.31.0-beta.1` a `beta.3`, saída de
`git ls-remote`). O fork está na 0.30. Uma migração futura do `storytold/winit` para 0.31 mudaria
a base de qualquer patch Android. Não verifiquei o que a 0.31 traz para caneta e IME no Android.

### 3.5 Onde fica a cola JNI sem violar a proibição de `unsafe`

Fatos:

1. O workspace tem `unsafe_code = "forbid"` (`Cargo.toml` raiz, seção `[workspace.lints.rust]`).
2. O `android_main` precisa de `#[unsafe(no_mangle)]`. Teste com rustc 1.97.0 e
   `#![forbid(unsafe_code)]`: "error: declaration of a `no_mangle` function".
3. O crate `jni` exige `unsafe` para obter a VM a partir do ponteiro que o `android-activity`
   entrega: `JavaVM::from_raw` é `pub unsafe fn` no `jni` 0.21.1 (`src/wrapper/java_vm/vm.rs:239`)
   e no 0.22.4 (`src/vm/java_vm.rs:422`); `AndroidApp::vm_as_ptr` devolve `*mut c_void`
   (`android-activity-0.6.1/src/lib.rs:810`).
4. O SAF devolve o resultado em `onActivityResult`. A NativeActivity não tem código Java do app
   para receber isso; o GameActivity é uma classe Java que o app estende. Inferido da arquitetura
   das duas; não verificado com código.
5. Há precedente no próprio WordCraft: o `fmv-macos-events` é um crate externo e auditado que
   encapsula o `unsafe` do Objective-C atrás de uma API segura, fixado com `=0.1.0`
   (`apps/wordcraft/Cargo.toml:33-37`).

Opções:

| Opção | Como | Prós | Contras |
|---|---|---|---|
| 1. Crate externo auditado (recomendada) | `craft-android` (nome sugerido) fora do workspace, com `jni`, `android_main` reexportado e API segura: `abrir_documento()`, `salvar_documento()`, `clipboard`, `imprimir_pdf()`, `falar()` | segue o precedente do `fmv-macos-events`; o workspace continua com `forbid` | um repositório a mais para manter e auditar |
| 2. Crate no workspace com lints próprios | omitir `[lints] workspace = true` só nele | simples | contraria o `CLAUDE.md` ("new crates ... add `[lints]` with `workspace = true`"); decisão dos mantenedores |
| 3. Classe Java/Kotlin que chama Rust só por dados | Activity em Kotlin faz o SAF e passa bytes ao Rust por JNI com funções exportadas | menos `unsafe` em Rust | ainda precisa de `no_mangle`; e contraria "Rust only" do `CLAUDE.md` (que fala de JS/TS, mas o espírito é só Rust). Decisão dos mantenedores |

Com GameActivity, alguma classe Java/Kotlin do app é inevitável (fato 4). Isso precisa ser
aceito pelos mantenedores antes de qualquer port.

## 4. Fase 2: compilação cruzada

Comando: `cargo check --target aarch64-linux-android -p <pacote>`, com `CARGO_TARGET_DIR`
separado. Sem NDK (seção 2).

| Ordem | Pacote | Resultado | Tempo | Log | Classe da falha |
|---|---|---|---|---|---|
| 1 | `wordcraft-doc` | passa | 29 s | `check-doc.log` | |
| 2 | `wordcraft-engine` | passa | 63 s | `check-engine.log` | |
| 3 | `wordcraft-layout` | passa | 16 s | `check-layout.log` | |
| 4 | `wordcraft-docx` | passa | 11 s | `check-docx.log` | |
| 5 | `wordcraft-pdf` | passa | 11 s | `check-pdf.log` | |
| 6 | `wordcraft-render` | passa | 9 s | `check-render.log` | |
| 7 | `wordcraft-cli` | passa | 1 s (dependências já checadas) | `check-wordcraft-cli.log` | |
| 8 | `wordcraft-ui-egui` | falha no `ring` (build script C: "failed to find tool aarch64-linux-android-clang") | 16 s | `check-wordcraft-ui-egui.log` | ambiente (falta NDK). Com NDK, o `ring` costuma compilar para Android; não verificado. Mesmo assim, **trivial (cfg)**: retirar o `craft-update` no Android |
| 8b | `wordcraft-ui-egui` sem `craft-update` | passa | 28 s | `exp-ui-egui.log` | |
| 9 | `wordcraft` (app desktop) | falha no `ring` | 76 s | `check-wordcraft.log` | como o 8 |
| 9b | `wordcraft` sem `craft-update` | falha: "Either game-activity or native-activity must be enabled" no `android-activity` | 19 s | `exp-wordcraft.log` | **trivial (cfg)**: ligar o feature |
| 9c | com `android-native-activity` | falha: "`accesskit` feature is only available with `android-game-activity`" | 30 s | `exp2-wordcraft.log` | **estrutural**: escolha de atividade (3.5) |
| 9d | com `android-game-activity` | falha: falta `aarch64-linux-android-clang++` (o GameActivity compila C++) | 4 s | `exp3-wordcraft.log` | ambiente (falta NDK) |
| 9e | NativeActivity sem `accesskit` | falha em `rfd` (12 erros) e `arboard` (8 erros) | 8 s | `exp4-wordcraft.log` | **moderada**: substituir os crates por código JNI próprio |

Os avisos nos logs 1 a 7 são só "patch ... was not used in the crate graph" (quatro patches de
crates do egui que esses pacotes não usam).

Aviso: `cargo check` não prova que o binário liga nem que roda. Ligação e execução ficam para a
sessão local com NDK.

## 5. Fase 3: protótipo (estado)

Repositório `geocida/wordcraft-android-probe`, commit `0528e3b`:

1. App eframe mínimo com as mesmas revisões de egui, eframe e winit que o WordCraft fixa
   (`Cargo.toml` do protótipo, `[patch.crates-io]`), fora do workspace (seção 3.5, fato 2).
2. Três modos num APK: `egui-wgpu`, `egui-glow` e `bruto`. O `bruto` lê o `MotionEvent` direto do
   `android-activity` (ferramenta, pressão, inclinação, orientação, distância, botões da caneta,
   histórico), para separar o que o hardware entrega do que o caminho winit e egui perde.
3. Projeto Gradle sem código Java (NativeActivity), scripts de build, instalação por adb via Wi-Fi,
   troca de modo e coleta de logcat; página `web/sonda.html` para medir o Chrome; roteiro com
   tabelas em `docs/roteiro-medicao.md`.

Verificado: `cargo check` e `cargo clippy` para `aarch64-linux-android` passam sem avisos; uma só
versão de `android-activity` (0.6.1) no grafo (`cargo tree -i android-activity`).
Não verificado: ligação com o NDK, build do APK, instalação e todas as medições.

Como o protótipo usa NativeActivity, ele mede o caminho mais simples. Se o T2 falhar (sem
composição), vale um segundo protótipo com GameActivity e um patch no winit para o `TextEvent`,
antes de decidir.

## 6. Tabela de bloqueios

Gravidade: **trivial** (cfg), **moderada** (substituir crate ou escrever código de plataforma
contido), **estrutural** (arquitetura, governança ou fork de terceiros). Horas: estimativa de
julgamento, inclui testes no aparelho.

| # | Item | Evidência | Gravidade | Solução proposta | Horas |
|---|---|---|---|---|---|
| 1 | `craft-update` puxa `ring` (C) e não tem função no Android | `check-wordcraft-ui-egui.log`; `exp-ui-egui.log` | trivial | cfg `not(any(wasm32, android))` no `ui-egui` (experimento já feito, ver patch) | 2 a 4 |
| 2 | Bloco `all(unix, not(macos))` puxa `winit/x11`, `zbus`, `wl-clipboard-rs` | `apps/wordcraft/Cargo.toml:52-60` | trivial | trocar por `all(unix, not(any(macos, android)))` nos manifestos e nos 14 blocos de código | 4 a 8 |
| 3 | `main.rs` usa `winit::platform::x11` | `apps/wordcraft/src/main.rs:462-468`; `winit/src/platform/mod.rs:21-22` | trivial | cfg; inferido, não compilado | incluído no 2 |
| 4 | Ponto de entrada Android e `cdylib` | teste `no_mangle` (3.5, fato 2) | estrutural (governança) | novo crate `apps/wordcraft-android` ou crate externo (3.5, opção 1); projeto Gradle | 8 a 16 |
| 5 | NativeActivity ou GameActivity; `accesskit` só no GameActivity | `exp2-wordcraft.log`; `eframe/src/lib.rs:153` | estrutural | GameActivity (precisa de classe Java/Kotlin e da biblioteca do GameActivity) | 8 a 16 |
| 6 | IME sem composição no winit Android | `winit/src/platform_impl/android/mod.rs:916-931, 477` | **estrutural, decisivo** | patch no `storytold/winit`: traduzir `TextEvent` do GameTextInput em `Ime::Preedit`/`Ime::Commit`, posição do cursor e texto ao redor; ajustar o `egui-winit` | 40 a 80 |
| 7 | `rfd` sem Android | `exp4-wordcraft.log` | moderada | SAF por JNI; abrir e salvar por bytes (`io.rs:146`, `:417`); lista de recentes com URIs persistentes | 24 a 40 |
| 8 | `arboard` sem Android; egui-winit sem clipboard no Android | `exp4-wordcraft.log`; `egui-winit/Cargo.toml:135-136` | moderada | `ClipboardManager` por JNI (texto e HTML; imagem depois) | 8 a 16 |
| 9 | Fontes do sistema | `crates/fonts/src/fontdb/names.rs:230-238` | trivial | acrescentar `/system/fonts`; testar substituições de fontes do Word | 2 a 4 |
| 10 | Dicionários Hunspell do sistema | `craft-proof/src/hunspell.rs:125-128` | trivial | embutir craft-i18n no build Android | 2 a 4 |
| 11 | Impressão (CUPS) | `craft-print/src/lib.rs:299-330, 516-530` | moderada | `PrintManager` por JNI ou exportar PDF e compartilhar | 8 a 16 |
| 12 | Leitura em voz alta (`espeak-ng`) | `crates/engine/src/speech.rs:877` | moderada | TextToSpeech por JNI, ou desligar | 4 a 8 |
| 13 | Uma janela por documento (`craft-window`) | compila (`dep-craft-window.log`); comportamento não verificado | moderada (não verificado) | uma janela com troca de documento no Android | 8 a 16 |
| 14 | Caneta: hover, botões, ferramenta, inclinação | `mod.rs:388`; `event.rs:849-865` | estrutural | patch no winit e no egui (seção 7) | 24 a 48 |
| 15 | Tinta sem pressão | `ink.rs:82`; `freeform.rs:71-72`; `draw.rs:431` | moderada | modelo de ponto com pressão e escrita no InkML (seção 7) | 16 a 32 |
| 16 | Ciclo de vida (segundo plano, rotação, tela dividida) | não verificado (roteiro, L1 a L7) | não verificado | depende do resultado no aparelho | não estimado |
| 17 | Interface feita para mouse (faixa de opções, alvos pequenos) | não verificado | não verificado | decisão de produto | não estimado |
| 18 | NDK indisponível nesta sessão | 403 em `dl.google.com` | ambiente | sessão local no WSL | 0 |

## 7. S Pen

Tudo nesta seção é **leitura de código**. Nada foi medido no aparelho; o roteiro do protótipo
(P1 a P12) mede cada item.

### 7.1 O que deve funcionar sem patch

1. **Pressão até o egui.** O winit põe `Force::Normalized(pointer.pressure())` em cada
   `Touch` (`mod.rs:418`), e o egui-winit repassa como `egui::Event::Touch { force }`
   (`egui-winit/src/lib.rs:2950-2968` do fork). Não verificado.
2. **Desenhar com a caneta como arraste.** O egui-winit emula o ponteiro primário a partir do
   primeiro toque (`lib.rs:2974-2991`), e a tinta do WordCraft usa esse ponteiro (`ink.rs:82`).
   Não verificado.

### 7.2 O que exige patch no winit e no egui

| Dado | winit Android hoje | egui hoje | Patch |
|---|---|---|---|
| Hover (ponta perto da tela) | `HoverEnter/Move/Exit` caem em `None // TODO mouse events` (`mod.rs:388`) | receberia `PointerMoved` sem botão | winit: emitir `CursorMoved`/`CursorEntered`/`CursorLeft` para a caneta |
| Botão lateral | `ButtonPress/Release` caem no mesmo `None` | `PointerButton::Secondary` existiria | winit: emitir `MouseInput` com o botão da caneta |
| Ferramenta (caneta, dedo, borracha) | não exposta; `Touch` não tem campo (`event.rs:849-865`) | `Event::Touch` não tem campo (`egui/src/data/input/event.rs:158-177`) | winit e egui: novo campo, ou um evento de caneta à parte |
| Inclinação e orientação | não expostas | sem campo | idem |
| Rejeição de palma | sem tipo de ferramenta, não dá para filtrar | idem | depende da linha acima |
| Pontos agrupados (histórico) | só o ponto atual de cada `MotionEvent` (`mod.rs:392-401`) | idem | winit: emitir o `history()` do `android-activity` |

O `android-activity` 0.6.1 já expõe tudo isso (`tool_type()`, `axis_value(Axis::Tilt)`,
`axis_value(Axis::Distance)`, `button_state().stylus_primary()`, `history()`;
`src/native_activity/input.rs:150, 250, 269, 276`). O trabalho é de tradução, não de acesso ao
hardware.

### 7.3 O que o WordCraft precisaria mudar

1. **Modelo de dados.** O traço guarda pontos `[f32; 2]` (`crates/doc/src/freeform.rs:71-72`) e
   uma largura por traço (`crates/engine/src/cmd/draw.rs:431`). Pressão por ponto pede
   `[f32; 3]` ou um vetor paralelo, com limite (`MAX_POINTS`, `freeform.rs:7`) e tratamento de
   valores hostis como manda o `CLAUDE.md`.
2. **Comando.** `draw.stroke` recebe `points` e `width` (`draw.rs:431-435`); precisaria aceitar
   pressão e manter compatibilidade com quem chama pelo MCP e pelo CLI.
3. **Entrada.** `ink.rs:82` lê o ponteiro emulado; teria de ler `Event::Touch` (que traz força)
   e ignorar toques de dedo enquanto a caneta desenha.
4. **Desenho.** `ink.rs:237-241` pinta com `Stroke::new(width, color)` fixo; precisaria de
   largura variável por segmento.
5. **Arquivo.** O leitor de DOCX já reconhece tinta como `application/inkml+xml` em
   `w14:contentPart` (`crates/docx/src/read/story/drawing.rs:36-39`). Não verifiquei se o escritor
   grava tinta como InkML nem se o InkML lido traz pressão.

## 8. Custo de manter um fork

Medido no histórico de `storytold/wordcraft` (`git log upstream/main`, 3.000 commits buscados,
o mais antigo de 2026-10-05):

| Medida | Valor (2026-10-19 a 2026-11-02) |
|---|---|
| PRs integrados (merges) | 368 |
| Commits | 1.367 |
| Pico de merges num dia | 194 (2026-10-22) |
| Arquivos tocados em `apps/wordcraft`, `crates/ui-egui` e `Cargo.toml` | 245 |
| Commits que mexeram no `Cargo.toml` raiz | 48 |
| Trocas da revisão de egui/winit | 1 (`fd071202`, 2026-10-24, migração para egui 0.37) |

Leitura:

1. As mudanças triviais (cfg) espalham-se por `apps/wordcraft` e `crates/ui-egui`, justamente as
   áreas de maior rotatividade. Mantidas num fork, gerariam conflito quase diário. Não medi
   conflitos de fato; é inferência pelo volume.
2. O patch de IME e caneta no winit é um fork de um fork: o `storytold/winit` já carrega 37
   commits sobre a 0.30.13 e documenta cada patch (W1 a W27). Um patch Android teria de entrar
   nessa lista ou ser rebaseado a cada troca de revisão.
3. **Estratégia que barateia:** em vez de fork, propor upstream em PRs pequenos, nesta ordem:
   (a) cfgs que excluem o Android dos blocos de desktop, inofensivos para as outras plataformas;
   (b) `craft-update` fora do Android; (c) um crate `apps/wordcraft-android` isolado; (d) patches
   de winit como W28 em diante. O `CLAUDE.md` exige `cargo xtask ci` antes de cada commit e
   proíbe mexer em CI em PR de funcionalidade; um alvo Android sem CI tende a quebrar sem aviso.
4. Estimativa (julgamento): mantendo fork próprio, 4 a 8 horas por semana só para acompanhar o
   upstream; com as mudanças aceitas no upstream, perto de zero para as triviais e algumas horas
   por troca de revisão do winit.

## 9. Comparação com o wordcraft-web no Chrome do tablet

| Item | Nativo (winit Android, pela leitura) | Web (eframe web do fork, pela leitura) | Fonte |
|---|---|---|---|
| Esforço para testar | Fases A e B (seção 1) | zero: `trunk serve` e `adb reverse` | `README.md:167`; `apps/wordcraft-web/Trunk.toml` |
| Pressão | chega ao egui | só por `Touch.force` nos eventos de toque; o `PointerEvent` não é lido para pressão | `eframe/src/web/events.rs:545-600`; `eframe/src/web/input.rs:98-103` |
| Inclinação, hover, botão lateral, ferramenta | não chegam | não são lidos (`pointerType`, `tiltX`, `tiltY` ausentes) | idem |
| Composição de texto (acentos, autocorreção, ditado) | ausente | há tratamento de `compositionstart`, `input` e `compositionend` | `eframe/src/web/text_agent.rs:465-482` do fork |
| Abrir e salvar arquivos | SAF por JNI (a escrever) | o que o navegador oferecer; não verificado no Chrome Android | não verificado |
| Funcionar sem rede | sim | não verificado (`grep serviceWorker` em `apps/wordcraft-web` não encontrou nada) | não verificado |
| Desempenho de desenho | não medido | não medido | roteiro D1 a D3 |

Testes no aparelho para os mesmos itens da Fase 3: roteiro, seção 6 (B1 a B11). A
`web/sonda.html` do protótipo mede o que o Chrome entrega; o `wordcraft-web` mede o que o
WordCraft aproveita.

Leitura provisória: a web no Chrome deve ganhar no texto, que é o essencial, e empatar ou perder
na caneta. Se o B7 confirmar que acentos, autocorreção e ditado funcionam no `wordcraft-web`, o
caminho de menor custo para o S Pen passa a ser ler `pressure` e `pointerType` no
`eframe/src/web/events.rs` (patch no fork do egui), não o port nativo. Estimativa (julgamento):
8 a 16 horas no eframe web, mais a seção 7.3 no WordCraft.

## 10. Pendências

1. Rodar a Fase 3 no aparelho (sessão local no WSL), preencher o roteiro e trazer os logs de
   `medicoes/` para cá.
2. Com NDK: ligar a `libprobe.so`, gerar o APK e confirmar que o `ring` compila para Android
   (se o `craft-update` for mantido em algum alvo Android, o que não recomendo).
3. Opcional (Fase 3, item 6): ponto de entrada Android para o `wordcraft-ui-egui` com o patch
   do experimento, documento de exemplo, rolagem e digitação.
4. Atualizar as seções 1, 6, 7 e 9 com os resultados e trocar "provisório" por definitivo.

## Apêndice: como reproduzir a Fase 2

```bash
rustup target add aarch64-linux-android
export CARGO_TARGET_DIR=~/dev/wc-target
for p in doc engine layout docx pdf render; do
  cargo check --target aarch64-linux-android -p wordcraft-$p
done
cargo check --target aarch64-linux-android -p wordcraft-cli
cargo check --target aarch64-linux-android -p wordcraft-ui-egui   # falha no ring sem NDK

# Experimento (numa cópia, não no branch):
git worktree add ../wc-exp HEAD --detach
cd ../wc-exp && git apply ../wordcraft/docs/android-eval/experimento-android.patch
cargo check --target aarch64-linux-android -p wordcraft-ui-egui   # passa
cargo check --target aarch64-linux-android -p wordcraft           # para em rfd e arboard
```
