# RepuLink — Spec: Escrow MVP en devnet

> **Para el agente (Claude Code).** Ejecuta las fases en orden. Cada task tiene su verificación: no avances si no está en verde. No re-abras decisiones de diseño ni agregues features fuera de alcance.

## Herramientas y forma de trabajo (leer antes de tocar código)

**MCP oficial de Solana — obligatorio.** Conectar el Solana Developer MCP antes de arrancar:

```bash
claude mcp add --transport http solana-mcp-server https://mcp.solana.com/mcp
```

Verificar con `/mcp` que expone: `Solana_Expert__Ask_For_Help`, `Solana_Documentation_Search`, `Ask_Solana_Anchor_Framework_Expert`, `list_sections`, `get_documentation`, `program_autofixer`. Reglas de uso:

1. Para cualquier duda de API/sintaxis de Anchor o Solana (eventos, CPI de token, constraints, cierre de ATAs), preguntar al MCP **antes** de escribir de memoria. La versión de Anchor del repo manda, no el conocimiento del modelo.
2. **Loop de autofixer:** después de escribir o modificar cualquier Rust del programa, pasar el archivo por `program_autofixer` (framework: `anchor`), aplicar los fixes sugeridos, y repetir hasta que `require_another_tool_call_after_fixing` sea `false`. Recién entonces compilar.
3. Ante un error de compilación o de test que no se resuelva en 2 intentos, consultar `Solana_Expert__Ask_For_Help` con el error completo y contexto — no iterar a ciegas.

**Subagentes (ya instalados como skills):** usar `scout` (read-only) para mapear el repo antes de Fase 0; usar `reviewer` al cerrar Fase 1, con el checklist de seguridad de los 5 bugs como criterio de revisión.

**Disciplina:** un commit por task, solo con su verificación en verde. No actualizar versiones de Anchor, Solana CLI, Rust ni dependencias del repo — si una versión bloquea algo, parar y preguntar.

## Objetivo

Implementar el ciclo completo **crear trabajo → depositar USDC en escrow → entregar → aprobar → liberar pago + emitir atestación**, en devnet, sobre un repo reestructurado y limpio. Este ciclo es el corazón del demo del 31 de agosto.

## Criterios de éxito globales

1. `anchor build` y `anchor test` en verde desde la raíz del repo reestructurado.
2. Ciclo happy path completo verificable en Solana Explorer (devnet): create → fund → deliver → release, con fee llegando a treasury y resto al freelancer.
3. Tests negativos de autorización pasan: ninguna wallet ajena puede operar un Job que no le corresponde.
4. Ningún dato PII o de contenido on-chain: solo pubkeys, montos, estados, timestamps y hashes.

## Decisiones cerradas (NO re-abrir)

1. **Custodia:** programa Anchor propio en devnet. Squads v4 queda para mainnet — fuera de alcance total.
2. **Modelo de confianza:** las pubkeys de `client`, `freelancer` y `arbiter` quedan grabadas en el `Job` al crearse. Solo esas wallets pueden ejecutar sus instrucciones respectivas. Esto reemplaza el modelo roto de badges (donde cualquiera aprobaba).
3. **Árbitro MVP:** una wallet de RepuLink guardada en una cuenta `Config` global, declarada públicamente. Nada de simular árbitros externos.
4. **Auto-release:** `review_window_secs` por Job, default **604800 (7 días)**. Referencia de mercado: Fiverr 3d, Upwork 14d.
5. **`mark_delivered` obligatorio** antes de poder liberar (habilita el timeout y da timestamps reales).
6. **Split on-chain/off-chain:** on-chain solo lo que necesita custodia o verificación (regla A/B/C). Brief, títulos, emails, chats, entregables → off-chain. On-chain va `terms_hash` y `delivery_hash` (SHA-256) como prueba verificable.
7. **Reutilizar, no reinventar.** El state machine del Job es lógica de negocio propia (no existe un escrow genérico reusable para esto — Squads es multisig de tesorería, no condicional con disputa). Todo lo demás se apoya en primitivas ya probadas: movimientos de USDC solo vía CPI al SPL Token Program oficial (`anchor_spl::token`), ATAs estándar (`anchor_spl::associated_token`), mint de USDC devnet existente (no crear mint propio). Cero lógica de transferencia o balances escrita a mano.

---

## Fase 0 — Reestructuración del repo + fixes baratos

Contexto: el diagnóstico previo (ya lo tienes) mapeó todas las rutas hardcodeadas. El escrow nuevo debe nacer en la estructura limpia.

**Task 0.1 — Mover `template_codespaces/` a la raíz.**
- Borrar primero el `package-lock.json` huérfano de la raíz (`"name": "Solana-Hackathon-WayLearn"`, packages vacío).
- Mover todo el contenido de `template_codespaces/` a la raíz del repo.
- Fusionar `.gitignore` raíz con el del template; subir `.prettierrc`/`.prettierignore`.
- Renombrar `"name"` en `package.json` a `repulink`.
- Verificación: `npm install && npm run build` en verde desde la raíz.

**Task 0.2 — Purga de dead code.**
- Borrar: `anchor/programs/repulink/src/tests.rs` (tests del vault del template), `src/components/VaultCard.tsx`, `src/generated/vault/`.
- Desinstalar deps nunca importadas: `@privy-io/react-auth`, `@privy-io/wagmi`, `helius-sdk`, `@metaplex-foundation/*`, `umi` (verificar con grep antes de borrar cada una; Privy se re-agregará en Fase 3 solo si se usa).
- Borrar `src/hooks/useReputationNFT.ts` y el botón que lo invoca en `ReputationCard.tsx` (construye una instrucción MPL Core inválida que nunca puede ejecutar).
- Verificación: `npm run build` y `cargo build-sbf` en verde; `grep -r` confirma cero referencias a lo borrado.

**Task 0.3 — Arreglar scripts de setup.**
- `.devcontainer/setup.sh` y `local-setup.sh`: eliminar el `npx create-solana-dapp` (recrearía el template encima del código). Reemplazar por instalación de toolchain + `npm install` del proyecto real.
- Actualizar `README.md`: rutas sin `template_codespaces/`, diagrama de estructura, ruta de la imagen de portada.

**Task 0.4 — Fixes al programa de badges existente (solo estos tres, nada más).**
1. `useOnChainData.ts:160` — memcmp: cambiar `badgeDiscriminator.toString("base64")` por `bs58.encode(badgeDiscriminator)`.
2. `create_badge` — agregar `require!` de longitud/no-vacío para `client_email` (≤128 bytes), consistente con las demás validaciones.
3. `close_profile` — agregar `require!(profile.badge_count == 0, ErrorCode::ProfileHasBadges)` para evitar el brickeo por re-inicialización.
- Verificación: `anchor test` de badges en verde; la lista de badges carga en el frontend.
- **NO** hacer más trabajo en el flujo de badges: no rediseñar autorización, no agregar close_badge, no tocar nada más. Ese flujo queda como legacy de perfil.

---

## Fase 1 — Programa de escrow (Anchor)

Nuevo módulo dentro del programa `repulink` existente (mismo program id, nuevas instrucciones y cuentas). Mantener el estilo del código existente: `checked_*` en toda aritmética, bumps guardados y reutilizados, `has_one` donde aplique.

Precisiones:

- **Token clásico:** USDC devnet es SPL Token clásico → `anchor_spl::token`. Nada de `token_interface` ni Token-2022 (es otro programa, fuera de alcance).
- **Mint configurable:** el mint llega por Config/env, nunca hardcodeado. En tests se crea un mint propio de 6 decimales.
- **Montos:** `u64` en unidades base (USDC = 6 decimales; la conversión es responsabilidad del frontend). El fee trunca hacia abajo (`checked_mul` → `checked_div`); el remanente queda del lado del freelancer — determinista y testeable con montos que no dividen exacto.
- **Eventos Anchor:** `emit!` en cada transición — `JobCreated`, `JobFunded`, `JobDelivered`, `JobReleased`, `JobRefunded`, `JobDisputed`, `JobResolved` — con pubkey del job, estado y timestamp. Es lo que después consumirá el indexado con webhooks de Helius: la frontera "quién indexa vs. quién calcula" del feedback del mentor queda resuelta desde el programa.

### Cuenta `Config` (singleton, PDA seeds = `[b"config"]`)

```rust
pub struct Config {
    pub admin: Pubkey,        // quien puede actualizar config
    pub arbiter: Pubkey,      // wallet RepuLink para disputas
    pub treasury: Pubkey,     // wallet que recibe fees (su ATA de USDC)
    pub fee_bps: u16,         // 150 = 1.5%. Se copia al Job al crearse
    pub bump: u8,
}
```

- `init_config(arbiter, treasury, fee_bps)` — solo una vez, admin = signer.
- `update_config(...)` — solo admin. `fee_bps` máximo 500 (5%), `require!`.

### Cuenta `Job` (PDA seeds = `[b"job", client.key(), job_id.to_le_bytes()]`)

`job_id: u64` lo genera el frontend (timestamp ms). Evita el contador con race condition que tiene el flujo de badges.

```rust
pub struct Job {
    pub job_id: u64,
    pub client: Pubkey,
    pub freelancer: Pubkey,
    pub arbiter: Pubkey,          // copiado de Config al crear
    pub mint: Pubkey,             // USDC devnet
    pub amount: u64,
    pub fee_bps: u16,             // copiado de Config al crear
    pub state: JobState,
    pub created_at: i64,
    pub funded_at: i64,
    pub delivered_at: i64,
    pub review_window_secs: u32,  // default 604800 (7 días), lo pasa el frontend
    pub terms_hash: [u8; 32],     // SHA-256 del brief acordado (se calcula off-chain)
    pub delivery_hash: [u8; 32],  // SHA-256 de la entrega, escrito en mark_delivered
    pub bump: u8,
}

pub enum JobState { Created, Funded, Delivered, Released, Refunded, Disputed, Resolved }
```

**Vault:** ATA de USDC cuyo owner es el PDA del Job. Los fondos solo se mueven por CPI firmada con los seeds del Job. Ninguna llave de RepuLink puede tocarlos.

### Instrucciones

| Instrucción | Signer | Transición | Efecto y constraints |
|---|---|---|---|
| `create_job(job_id, freelancer, amount, fee_bps_snapshot, terms_hash, review_window_secs)` | client | — → Created | `require!(freelancer != client)`; `require!(amount > 0)`; `review_window_secs` entre 86400 y 2592000 (1–30 días); copia `arbiter` y `fee_bps` desde Config |
| `fund_job` | client | Created → Funded | Transfer USDC client → vault ATA por `amount` exacto; `has_one = client`; sella `funded_at` |
| `mark_delivered(delivery_hash)` | freelancer | Funded → Delivered | `has_one = freelancer`; sella `delivered_at` |
| `approve_release` | client | Delivered → Released | fee = `amount * fee_bps / 10000` → treasury ATA; resto → freelancer ATA; toda aritmética `checked_*` |
| `claim_timeout` | freelancer | Delivered → Released | `require!(clock.unix_timestamp >= delivered_at + review_window_secs)`; mismo payout que approve_release |
| `cancel_refund` | client | Created o Funded → Refunded | Solo antes de `mark_delivered`. Si Funded: devolver todo al client, sin fee |
| `open_dispute` | client **o** freelancer | Funded o Delivered → Disputed | `require!(signer == job.client \|\| signer == job.freelancer)` |
| `resolve_dispute(freelancer_amount)` | arbiter | Disputed → Resolved | `require!(signer == job.arbiter)`; `require!(freelancer_amount <= amount)`; fee sobre `freelancer_amount` → treasury; `freelancer_amount - fee` → freelancer; resto → client |
| `close_job` | client | desde Released/Refunded/Resolved | Cierra Job y vault ATA (debe estar en 0); renta de vuelta al client |

### Seguridad obligatoria (bugs clásicos de Solana — no negociable)

Cada instrucción debe cumplir las 5, sin excepción:

1. **Ownership.** `Job`, `Config` y toda cuenta de estado son `Account<'info, T>`, nunca `UncheckedAccount`. Anchor valida owner + discriminador automáticamente; no reemplazar por checks manuales.
2. **Signer.** Toda instrucción que cambia estado exige `Signer<'info>` **y** `has_one` (o `require!(signer.key() == job.campo)`) sobre el campo correspondiente del Job. Nunca asumir permiso porque "alguien firmó".
3. **PDA.** Seeds explícitas y bump guardado (`seeds = [b"job", job.client.as_ref(), job.job_id.to_le_bytes().as_ref()], bump = job.bump`) en cada cuenta derivada. Nunca aceptar un Job/Vault como cuenta libre sin que Anchor la valide contra esas seeds.
4. **CPI.** `token_program`, `associated_token_program`, `system_program` siempre `Program<'info, X>` tipado — jamás `UncheckedAccount` ni pubkey libre. No se invoca ningún programa recibido como parámetro arbitrario.
5. **Matemática.** Todo cálculo de fee: `checked_mul` **antes** de `checked_div` (multiplicar primero preserva precisión), nunca dividir primero. Confirmar `overflow-checks = true` en `Cargo.toml` tras la reestructuración de Fase 0.

### Errores custom

`InvalidState`, `Unauthorized`, `ReviewWindowNotElapsed`, `InvalidAmount`, `InvalidReviewWindow`, `SelfDealingNotAllowed`, `MathOverflow`, `FeeTooHigh`, `ProfileHasBadges` (Fase 0).

### Tests (obligatorios, este es el estándar)

Migrar a **LiteSVM** — nada de tests acumulando estado en devnet. El test de timeout exige manipular el clock (`setClock`), que LiteSVM soporta y localnet no da fácil; si LiteSVM presenta problemas con SPL Token, consultar el MCP antes de degradar a otra estrategia. Cobertura mínima:

1. Happy path: create → fund → deliver → release; asserts de balances (fee exacto en treasury, resto en freelancer).
2. Timeout: deliver + warp del clock 7 días + `claim_timeout` exitoso; y el negativo (warp 6 días → falla con `ReviewWindowNotElapsed`).
3. Refund: fund → cancel_refund → balance completo de vuelta; y el negativo (tras deliver → falla).
4. Disputa: open_dispute por freelancer → resolve_dispute por arbiter con split 60/40 → asserts de los tres balances.
5. Autorización (los más importantes): wallet random no puede fund/deliver/release/resolve; el client no puede `mark_delivered`; el freelancer no puede `approve_release`; nadie que no sea arbiter puede `resolve_dispute`; `create_job` con freelancer == client falla; pasar un Job/Vault PDA de otro client o job_id distinto es rechazado por Anchor (test explícito de sustitución de PDA).
6. Estados: cada instrucción falla con `InvalidState` desde estados no permitidos (al menos release desde Created y deliver desde Created).

Verificación de fase: `anchor test` en verde con todos los casos; regenerar cliente Codama (`npm run codama` o el script equivalente) sin errores.

---

## Fase 2 — Atestación al liberar (SAS)

**Task 2.1 — Spike SAS (timebox: medio día).** Emitir una atestación de prueba con el SDK de Solana Attestation Service en devnet, firmada por una keypair "attestation authority" de RepuLink, referenciando un Job PDA (pubkey del job + estado final + timestamps en el payload del schema).
- Si el spike funciona: **Task 2.2a** — servicio backend mínimo (script Node, puede correr manual para el demo) que ante un Job en `Released`/`Resolved` emite la atestación SAS.
- Si SAS da guerra (docs/SDK inmaduros): **Task 2.2b (fallback)** — cuenta `Attestation` PDA propia escrita por la authority, con los mismos campos. Documentar la decisión en el README.
- Verificación: atestación visible/consultable en devnet, vinculada al Job del test end-to-end.

---

## Fase 3 — Frontend mínimo del flujo

**Task 3.1 — Unificar stack.** Todo en `@solana/kit` + cliente Codama generado (patrón de `useRepulink.ts`, que es el bueno). Eliminar `@solana/web3.js` y `@coral-xyz/anchor` del frontend; reemplazar la deserialización manual de `useOnChainData.ts` por los decoders de `src/generated/repulink/accounts/`. Un solo RPC (Helius via env, con validación al arrancar: si falta la var, error claro).

**Task 3.2 — Páginas del flujo escrow** (UI mínima funcional, sin pulir diseño):
- Crear trabajo (client): form → hash del brief con SubtleCrypto → `create_job` + `fund_job` **en una sola transacción** (dos instrucciones, atómico: o el job nace fondeado o no nace — sin limbo Created sin fondos).
- Vista del trabajo: estado actual, timestamps, countdown de la ventana de revisión, botones condicionales por rol conectado (deliver / release / dispute / refund).
- Usar `useParams` de react-router (no parsear `window.location` a mano).
- Errores mapeados a mensajes humanos (mínimo los errores custom del programa).

**Task 3.3 — Refetch antes de derivar PDAs** dependientes de estado (lección del bug de `badgeIndex` stale).

**Task 3.4 — Commitment level.** Usar `"confirmed"` para feedback rápido de UI en cualquier transacción. Para las que mueven fondos (`fund_job`, `approve_release`, `claim_timeout`, `resolve_dispute`, `cancel_refund`), esperar `"finalized"` antes de marcar la operación como exitosa o habilitar el siguiente paso — es dinero, aunque sea de prueba en devnet.

Verificación de fase: ciclo completo ejecutado desde la UI en devnet con dos wallets de prueba, links de Explorer capturados (sirven para el demo y el M5).

---

## Fuera de alcance — NO hacer

- Squads v4, mainnet, auditorías.
- Rediseñar el flujo de badges legacy (solo los 3 fixes de Fase 0).
- Trust Agent, RepuScore, indexado con webhooks de Helius (fase posterior, no de esta spec).
- Múltiples mints/tokens: solo USDC devnet.
- Sistema de notificaciones, emails, matching.
- Token-2022 / Token Extensions (otro programa; verificar su soporte no es de este MVP).
- Actualizar toolchain o dependencias del repo.
- Cualquier feature no listada. Si algo parece necesario y no está aquí, preguntar antes de implementar.
