# Tiered Agent Crew

**OpenAI Codex ve Claude Code için Kademeli Ajan ve Token Disiplini Yapılandırması** · Sıcak Önbellek (Warm Cache) · Fresh Context Judge · 5-Trigger Advisor · 360k Dengeli Bağlam

![Codex](https://img.shields.io/badge/OpenAI_Codex-CLI_%2F_App-0066CC)
![Claude Code](https://img.shields.io/badge/Claude_Code-2.1%2B-7C3AED)
![Lisans](https://img.shields.io/badge/Lisans-Apache_2.0-10B981)

Yapay zeka tabanlı kodlama ajanlarında (Codex ve Claude Code) kredi ve token limitlerinin hızla tükenmesinin dört temel nedeni vardır:

1. **Kontrolsüz Alt-Ajan Açılması (Sub-Agent Spam):** Açılan her alt-ajan, sistem talimatlarını ve MCP/eklenti araç şemalarını (`~25k–40k` token) sıfırdan ve önbelleksiz (uncached) olarak yeniden yükler.
2. **Yüksek Düşünme Eforu (`high` / `xhigh`):** Rutin dosya okuma ve düzenleme adımlarında kullanılan yüksek muhakeme eforu, görünmeyen düşünme (reasoning) tokenlarını 4–8 kat artırır.
3. **Kontrolsüz Bağlam Şişmesi ve Erken Sıkıştırma Tuzağı:** Çok dar bağlam sınırı (`<200k`) büyük projelerde görevin ortasında sıkıştırma tetikleyerek **Prompt Cache'i (%99 indirimi) sıfırlar** ve modelin unuttuğu dosyaları baştan okumasına yol açar; çok geniş sınır (`>500k`) ise modelin dikkatini dağıtır ve maliyeti katlar.
4. **Yerleşik Danışman ve Eklenti Şişkinliği:** Yerleşik danışman araçlarının tüm sohbet geçmişini baştan okuması ve kullanılmayan eklenti şemalarının her turda taşınması sabit maliyeti katlar.

**Tiered Agent Crew**, bu dört sorunu istemci tarafı kurallar (`AGENTS.md` / `CLAUDE.md`), kademeli alt-ajan tanımları ve bağlam sıkıştırma ayarlarıyla çözen hazır bir yapılandırma setidir.

---

## Temel Mimari İlkeler

- **Sıcak Önbellek (Warm Cache) Önceliği:** Ana oturum `%97–99` indirimli prompt önbelleğine sahiptir. Planlama, 1–2 dosyalık düzenlemeler, hedefli `grep` (`rg`) aramaları ve satır aralığı okumaları alt-ajan açılmadan doğrudan ana oturumda (`medium` efor) yapılır.
- **İzole ve Hedefli Alt-Ajanlar:** Alt-ajanlar yalnızca ana bağlamı kirletecek **10+ dosyalık geniş aramalarda** veya **3+ bağımsız modülü kapsayan paralel kod işlerinde** devreye girer.
- **5 Tetikleyicili Kriz Danışmanı (Advisor):** Tüm sohbeti yeniden okuyan yerleşik danışman kapatılır (`CLAUDE_CODE_DISABLE_ADVISOR_TOOL=1`). Danışman alt-ajanı yalnızca işçi aynı hatada iki kez takıldığında, maksimum **300 kelimelik yapılandırılmış özet (Brief) + 10 satırlık proje haritası** ile çağrılır.
- **Sıfır Hafızalı Denetçi (Fresh-Context Judge):** Pahalı `xhigh` eforu yerine, yapım sohbetinin önyargısını taşımayan sıfır hafızalı bir `medium` denetçi yalnızca `kabul kriteri + git diff + test sonucu` üçlüsünü denetler.
- **Dengeli Bağlam Sınırı (`450k` Pencere / `360k` Auto-Compact):** Büyük projelerde görevin ortasında hafıza kaybı ve önbellek bozulması (cache invalidation) yaşanmaması için `360k` eşiği kullanılır; `500k+` üstü dikkat dağılması ve aşırı şişme ise otomatik kesilir.

---

## İş Akışı Diyagramı

```text
                             [ Görev Başlangıcı ]
                                      │
                                      ▼
                     ┌─────────────────────────────────┐
                     │      Ana Oturum (Medium)        │
                     │  • 5-10 satırlık plan çıkarır   │
                     │  • 1-2 dosyalık işi kendi yapar │
                     │  • Hedefli rg / satır okur      │
                     └────────┬───────────────┬────────┘
                              │               │
         10+ Dosya Geniş Arama│               │ 3+ Bağımsız Modül İşi
                              ▼               ▼
                     ┌────────────────┐ ┌────────────────┐
                     │  Explore/Scout │ │ Worker/Builder │
                     │  Luna / Haiku  │ │  Sol / Sonnet  │
                     │     (low)      │ │    (medium)    │
                     │  dosya:satır + │ │ Kendi modülünü │
                     │  5 satır kesit │ │ yazar ve dener │
                     └────────┬───────┘ └────────┬───────┘
                              │                  │
                              └────────┬─────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    │ 2 Kez Aynı Hatada Takıldı mı?       │
                    ├─ Evet ──► [ Advisor (High) ]        │
                    │           (300 kelime Brief okur,   │
                    │            3 adımlık çözüm döner)   │
                    └─ Hayır / Çözüldü                    │
                                       │                  │
                                       ▼                  ▼
                     ┌─────────────────────────────────┐
                     │   Judge / Verifier (Fresh Ctx)  │
                     │  • Sadece büyük/kritik işlerde  │
                     │  • Spec + git diff + test okur  │
                     └─────────────────┬───────────────┘
                                       │
                                       ▼
                        [ Görsel İş Varsa: En Sonda 1 Kez ]
                        [     terra_visual_verifier       ]
                                       │
                                       ▼
                                  [ Tamamlandı ]
```

---

## Rol ve Model Kademeleri

### 1. OpenAI Codex Kademeleri

| Rol | Ajan / Konum | Model | Efor | Görev ve Kredi Kuralı |
|---|---|---|---|---|
| **Ana Yürütücü / Plan** | Ana Oturum | `gpt-6.1-sol` | `medium` *(Plan modunda `high`)* | `plan_mode_reasoning_effort = "high"` ile Plan modunda `high`, kodlamada `medium` çalışır. |
| **Geniş Arama (Scout)** | `sol_explorer` | `gpt-6-luna` | `medium` (`read-only`) | Sadece 10+ dosyalık repo taramalarında açılır; `dosya:satır` + maks. 5 satır kesit döner (`Sol` kotası harcamaz). |
| **Kod İşçisi (Builder)** | `sol_worker` | `gpt-6.1-sol` | `medium` (`workspace-write`) | 3+ bağımsız modüllü işlerde paralel açılır; sadece kendi dosyalarını düzenler ve test koşar. |
| **Kriz Danışmanı** | `sol_advisor` | `gpt-6.1-sol` | `high` (`read-only`) | Sadece 5 kriz tetikleyicisinde, 300 kelimelik Brief + 10 satırlık harita ile çağrılır. |
| **Kod Hakemi (Judge)** | `sol_verifier` | `gpt-6.1-sol` | `medium` (`read-only`) | Sıfır hafızayla (fresh context) sadece `spec + git diff + test` çıktısını denetler. |
| **Görsel İşçi** | `terra_asset_worker` | `gpt-6.1-terra` | `medium` (`workspace-write`) | Çoklu doku, sprite/asset üretimi ve görsel tutarlılık işlerini yürütür. |
| **Görsel Hakem** | `terra_visual_verifier` | `gpt-6.1-terra` | `medium` (`read-only`) | Ara adımlarda çalışmaz; tüm kod testleri geçtikten sonra **en sonda 1 kez** ekran görüntüsüne bakar. |

### 2. Claude Code Kademeleri

| Rol | Ajan / Konum | Model | Efor | Görev ve Kredi Kuralı |
|---|---|---|---|---|
| **Ana Yürütücü** | Ana Oturum | `sonnet` | `medium` | Planlama, 1–2 dosyalık işler ve doğrudan dosya okumaları (ağır mimaride `opusplan`). |
| **Geniş Arama (Scout)** | `Explore` | `haiku` | `low` | Yerleşik Explore ajanını ezer; salt-okunur `file:line` ve kısa kesit döndürür. |
| **Kod İşçisi (Builder)** | `worker` | `sonnet` | `medium` | Çoklu modül işlerinde kod yazar ve test koşar (`haiku` modeline asla kod yazdırılmaz). |
| **Kriz Danışmanı** | `advisor` | `opus` | `high` | Yerleşik advisor kapalıdır; 5 kriz tetikleyicisinde 300 kelimelik Brief ile çağrılır. |
| **Kod Hakemi (Judge)** | `checker` | `opus` | — | Sıfır hafızayla `git diff` ve test sonuçlarını denetler, kenar durumlarını kırmaya çalışır. |

---

## Depo Yapısı

```text
├── codex/                              # OpenAI Codex yapılandırması
│   ├── agents/                         # ~/.codex/agents/ altına kopyalanacak alt-ajanlar
│   │   ├── sol-explorer.toml           # Geniş arama filtresi (Luna, medium)
│   │   ├── sol-worker.toml             # Modül kod işçisi (Sol, medium)
│   │   ├── sol-advisor.toml            # 5 tetikleyicili kriz danışmanı (Sol, high)
│   │   ├── sol-verifier.toml           # Sıfır hafızalı diff denetçisi (Sol, medium)
│   │   ├── terra-asset-worker.toml     # Görsel/doku üretim işçisi (Terra, medium)
│   │   └── terra-visual-verifier.toml  # Son adım görsel denetçi (Terra, medium)
│   ├── AGENTS.md                       # ~/.codex/AGENTS.md global çalışma kuralları
│   └── config-snippets.toml            # ~/.codex/config.toml için model, compact ve eklenti ayarları
│
├── claude/                             # Claude Code yapılandırması
│   ├── agents/                         # ~/.claude/agents/ altına kopyalanacak alt-ajanlar
│   │   ├── Explore.md                  # Salt-okunur arama ajanı (haiku)
│   │   ├── worker.md                   # Kod üretim işçisi (sonnet)
│   │   ├── advisor.md                  # 5 tetikleyicili kriz danışmanı (opus)
│   │   └── checker.md                  # Yıkıcı test ve diff hakemi (opus)
│   ├── CLAUDE.md                       # ~/.claude/CLAUDE.md global ekip kuralları
│   └── settings-snippets.json          # ~/.claude/settings.json için model ve env ayarları
│
└── README.md
```

---

## Kurulum

### 1. OpenAI Codex Kurulumu

**PowerShell (Windows):**
```powershell
New-Item -ItemType Directory -Force "$HOME\.codex\agents" | Out-Null
Copy-Item ".\codex\agents\*.toml" "$HOME\.codex\agents\" -Force
Copy-Item ".\codex\AGENTS.md" "$HOME\.codex\AGENTS.md" -Force
```

Ardından [`codex/config-snippets.toml`](./codex/config-snippets.toml) içindeki ayarları `~/.codex/config.toml` dosyanıza ekleyin:
- Ana modeli `medium` efora, varsayılan alt-ajanı `Luna` (`low`) katmanına sabitler.
- Bağlam sınırını `450000` / otomatik sıkıştırmayı `360000` token olarak ayarlar (büyük projelerde önbelleği bozmadan tam hafıza korur).
- Kodlamada kullanılmayan ağır eklentileri (`spreadsheets`, `presentations`, `pdf`, `documents`, `template-creator`, `visualize`) devre dışı bırakır.

### 2. Claude Code Kurulumu

**PowerShell (Windows):**
```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\agents" | Out-Null
Copy-Item ".\claude\agents\*.md" "$HOME\.claude\agents\" -Force
Copy-Item ".\claude\CLAUDE.md" "$HOME\.claude\CLAUDE.md" -Force
```

Ardından [`claude/settings-snippets.json`](./claude/settings-snippets.json) içindeki anahtarları `~/.claude/settings.json` dosyanızla birleştirin (`model: "sonnet"`, `CLAUDE_CODE_SUBAGENT_MODEL: "haiku"`, `CLAUDE_CODE_DISABLE_ADVISOR_TOOL: "1"`).

---

## Kriz Danışmanı (Advisor) Kullanım Protokolü

`sol_advisor` (Codex) ve `advisor` (Claude Code) **yalnızca şu 5 durumda** çağrılır:
1. Aynı hata 2 dürüst düzeltme denemesinden sonra devam ediyorsa.
2. Mimari, veri modeli veya kütüphane seçimi yapılması gerekiyorsa.
3. Değişiklik kimlik doğrulama (auth), ödeme, migration veya kullanıcı verisine dokunuyorsa.
4. Testler geçmesine rağmen çalışma zamanı davranışı hâlâ hatalıysa.
5. Düzeltme 5'ten fazla dosyayı veya dışa açık bir arayüzü (public API) değiştirecekse.

**Danışmana Gönderilecek Brief Formatı (Maksimum 300 Kelime):**
> Sohbet geçmişi asla yapıştırılmaz; yalnızca aşağıdaki şablon gönderilir:
> - **Goal:** Tek cümlede hedeflenen davranış
> - **Map:** 10 satırlık proje haritası
> - **Tried:** Denenen 2 yaklaşım ve neden başarısız olduğu
> - **Evidence:** Tam hata çıktısı veya beklenen vs. gerçekleşen durum
> - **Files:** İlgili `dosya:satır` aralıkları (en fazla 5 dosya)
> - **Options:** Arasında kalınan 2 teknik seçenek
> - **Question:** Karar verilmesi gereken tek net soru

---

## Sorun Giderme

| Belirti | Olası Neden | Çözüm |
|---|---|---|
| Kredi/limit çok hızlı tükeniyor | Kullanılmayan eklentiler/MCP'ler açık veya her küçük iş için alt-ajan açılıyor | `~/.codex/config.toml` içindeki ofis eklentilerini `enabled = false` yapın; 1–2 dosyalık işlerde alt-ajan açmayın |
| Bağlam çubuğu 500k+ üstüne çıkıyor veya çok erken sıkışıyor | `model_auto_compact_token_limit` dengesiz tanımlanmış | `config.toml` içinde değeri `360000` (pencereyi `450000`) yapın |
| Yeni eklenen alt-ajanlar görünmüyor | Uygulama açıkken dosya eklendi | Codex veya Claude Code'u yeniden başlatın |
| Claude yerleşik advisor kredisi uyarısı veriyor | Yerleşik advisor aracı açık kalmış | `settings.json` içinde `"CLAUDE_CODE_DISABLE_ADVISOR_TOOL": "1"` ayarının bulunduğundan emin olun |

---

## Lisans

Bu proje **Apache-2.0** lisansı altında sunulmaktadır.
