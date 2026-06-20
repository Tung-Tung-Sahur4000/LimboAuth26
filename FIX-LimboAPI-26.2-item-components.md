# 🛠️ Handoff: Fix `unsupported version: 26.2` in LimboAPI (and verify LimboAuth/LimboFilter)

> **TL;DR** — LimboAPI's Velocity 26.2 update is **incomplete**. The protocol max and a few
> `>= MINECRAFT_26_1` checks were updated, but the **Minecraft data generator's version list was
> never extended to 26.2**, so every table that is keyed by *generated per-version data*
> (item data-components, item IDs, block states, block entities, tags) has entries only up to
> **26.1**. The moment a 26.2 client triggers an item/inventory packet, LimboAPI throws.
> **LimboAuth is NOT the cause** — it has no protocol code and builds/loads fine.

---

## 1. Symptom

Server: Velocity `3.5.0-SNAPSHOT` (VeloFlame fork), with `limboapi 1.1.27-SNAPSHOT`,
`limboauth 1.1.14`, `limbofilter 1.1.19`. All plugins **load** fine. The crash happens when
LimboFilter builds its captcha inventory (a `SetSlot` packet carrying item components):

```
[ERROR] [limbofilter] Couldn't pass ProxyInitializeEvent to limbofilter 1.1.19
net.elytrium.commons.utils.reflection.ReflectionException: ...
    at ...fastprepare.PreparedPacketFactory.encodeId(PreparedPacketFactory.java:185)
    at net.elytrium.limbofilter.cache.CachedPackets.createCaptchaAttemptsPacket(CachedPackets.java:177)
    ...
Caused by: java.lang.IllegalArgumentException: unsupported version: 26.2
    at net.elytrium.limboapi.server.item.SimpleItemComponentManager.getId(SimpleItemComponentManager.java:151)
    at net.elytrium.limboapi.server.item.SimpleItemComponentMap.write(SimpleItemComponentMap.java:75)
    at net.elytrium.limboapi.protocol.packets.s2c.SetSlotPacket.encodeModern(SetSlotPacket.java:85)
```

LimboFilter is just the *trigger*. The throw is entirely inside **LimboAPI**.

---

## 2. Root cause (exact)

### 2a. The throwing code
`plugin/src/main/java/net/elytrium/limboapi/server/item/SimpleItemComponentManager.java`

```java
private static final Map<ProtocolVersion, Object2IntMap<String>> ID = new HashMap<>();

public int getId(String name, ProtocolVersion version) {
  Object2IntMap<String> ids = ID.get(version);
  if (ids == null) {
    throw new IllegalArgumentException("unsupported version: " + version);   // <-- here, for 26.2
  }
  ...
}
```

`ID` is populated in a `static {}` block from a **bundled resource**
`/mapping/data_component_types_mapping.json`, which maps each component →
`{ versionName -> protocolId }`. A version only lands in `ID` if that JSON contains its version
string (e.g. `"26.2"`). It does not — it only goes up to `"26.1"`.

### 2b. Why the JSON has no 26.2
That JSON is **generated**, not checked in. It is produced by LimboAPI's Minecraft data generator,
which iterates the build-script `MinecraftVersion` enum in **`plugin/build.gradle`**. That enum
ends at 26.1:

```groovy
  MINECRAFT_1_21_9(773),
  MINECRAFT_1_21_11(774),
  MINECRAFT_26_1(774)          // <-- LAST entry. No MINECRAFT_26_2.

  public static final List<MinecraftVersion> WORLD_VERSIONS = List.of(
    ...
    MINECRAFT_1_21_11,
    MINECRAFT_26_1               // <-- also stops at 26.1
  )

  public static final MinecraftVersion MAXIMUM_VERSION = values()[values().length - 1]
```

`gradle.properties` also pins `gameVersion=26.1`.

Because the generator never iterates a `MINECRAFT_26_2` entry, **none** of the generated
per-version mappings include a `26.2` key.

### 2c. What was done vs. what was missed in the 26.2 update
- ✅ Done: `SUPPORTED_MAXIMUM_PROTOCOL_VERSION_NUMBER 775 → 776`
- ✅ Done: open-ended `BlockEntityVersion` / `WorldVersion` ranges `EnumSet.range(MINECRAFT_26_1, MAXIMUM_VERSION)`
- ✅ Done: packet registrations are already open-ended — `createMapping(..., MINECRAFT_26_1, true)`
  in `LimboProtocol.java` covers 26.2 (packet IDs are identical to 26.1).
- ❌ **Missed:** the **data generator** `MinecraftVersion` enum + `WORLD_VERSIONS` (and
  `gameVersion`) were not extended to 26.2, so all generated per-version data stops at 26.1.

---

## 3. Other tables with the same 26.1-only gap

The item-component throw is the first one hit, but every map keyed by *generated, exact-version*
data has the same hole. Fixing the generator (Option A) fixes all of them at once. For awareness:

| File | Lookup | Failure mode for 26.2 |
|---|---|---|
| `server/item/SimpleItemComponentManager.java:149` | `ID.get(version)` | `IllegalArgumentException: unsupported version` (the reported crash) |
| `server/world/SimpleItem.java:55` | `versionIDs.get(version)` | `null` item id → NPE on item write |
| `server/world/SimpleBlock.java:229` | `MODERN_BLOCK_STATE_IDS_MAP.get(version)` | `null` → NPE on chunk/block write |
| `server/world/SimpleBlockEntity.java:54` | `versionIDs.get(version)` | `null` block-entity id |
| `server/world/SimpleTagManager.java:71` | `VERSION_MAP.get(version)` | `null` tags |
| `protocol/util/NetworkSection.java:95/147` | `storages.get(version)` | `null` chunk storage |
| `protocol/LimboProtocol.java:154` | `playProtocolRegistryVersions.get(version)` | possible `null` registry |

> NOTE: pure `>= MINECRAFT_26_1` **comparisons** (lots of them in `LimboImpl`, `ChunkDataPacket`,
> `NetworkSection`, `TimeUpdatePacket`) are already correct for 26.2 — `26.2 >= 26.1` is true.
> Only the **exact-version map lookups** above are broken.

---

## 4. Fix — Option A (RECOMMENDED: regenerate data for 26.2)

This is the complete fix and resolves every table in §3 in one shot.

1. **Add 26.2 to the generator enum** in `plugin/build.gradle`:
   ```groovy
     MINECRAFT_1_21_11(774),
     MINECRAFT_26_1(774),
     MINECRAFT_26_2(<26.2 data/world version number>)   // add this
   ```
   The constructor arg is the version's data/world version (same kind of value `MINECRAFT_26_1`
   uses — note 26.1 shares `774` with 1.21.11). Use the correct value for 26.2. The generator
   derives the Mojang version string as `name -> "26.2"` and downloads that server jar via
   `manifestUrl` (`version_manifest.json`), so Mojang's manifest must list `26.2`.

2. **Add it to `WORLD_VERSIONS`** in the same enum *only if* 26.2 introduced new world/block data.
   (If 26.2 is block-identical to 26.1, you may leave `WORLD_VERSIONS` ending at 26.1; verify.)

3. **Bump** `gradle.properties`: `gameVersion=26.2` (cache-invalidation marker for CI).

4. **Rebuild on JDK 25** so the data generator runs:
   ```bash
   JAVA_HOME=<jdk25> ./gradlew :plugin:build         # runs generateMappings against the 26.2 jar
   JAVA_HOME=<jdk25> ./gradlew publishToMavenLocal   # if LimboAuth/Filter resolve from mavenLocal
   ```
   - The generator needs **JDK 25** (MC 26.x server jars are Java 25 / class file 69) and **network
     to Mojang** (`launchermeta.mojang.com` + piston-data) to download the 26.2 server jar.
   - Behind a TLS-intercepting proxy, point the JVM at the system truststore to avoid
     `PKIX path building failed`:
     ```bash
     JAVA_TOOL_OPTIONS="-Djavax.net.ssl.trustStore=/usr/lib/jvm/java-21-openjdk-amd64/lib/security/cacerts -Djavax.net.ssl.trustStorePassword=changeit"
     ```

5. Ship the regenerated `limboapi` jar. The new `data_component_types_mapping.json` (and the other
   generated mappings) will contain `26.2`, and `getId` resolves it.

---

## 5. Fix — Option B (fast code fallback, no regeneration)

Use this if you can't run the generator quickly. Since 26.2 is packet/registry-identical to 26.1,
fall back to the nearest lower supported version when an exact entry is missing. **You must apply
this consistently to every map in §3, or you'll whack-a-mole** through item → block → tag NPEs.

Example for `SimpleItemComponentManager` (in the `static {}` block, after `ID` is built — alias
any newer protocol version to the highest version actually present):

```java
// After populating ID from the generated JSON:
for (ProtocolVersion version : ProtocolVersion.values()) {
  if (version.compareTo(ProtocolVersion.MINECRAFT_1_20_5) < 0 || ID.containsKey(version)) {
    continue;
  }
  // Inherit the table of the highest known version <= this one (e.g. 26.2 -> 26.1).
  ProtocolVersion fallback = null;
  for (ProtocolVersion candidate : ID.keySet()) {
    if (candidate.compareTo(version) <= 0 && (fallback == null || candidate.compareTo(fallback) > 0)) {
      fallback = candidate;
    }
  }
  if (fallback != null) {
    ID.put(version, ID.get(fallback));
  }
}
```

Apply the equivalent fallback to `SimpleItem.versionIDs`, `SimpleBlock.MODERN_BLOCK_STATE_IDS_MAP`,
`SimpleBlockEntity.versionIDs`, `SimpleTagManager.VERSION_MAP`, `NetworkSection` storages, and
`LimboProtocol.playProtocolRegistryVersions`.

> Trade-off: B is a behavioral shim that assumes 26.2 == 26.1 data. Correct *today* (no new
> packets), but A is the durable answer and future-proof for the next MC bump.

---

## 6. Validation

- `./gradlew :plugin:build` green on JDK 25.
- Drop the rebuilt `limboapi` jar + existing `limboauth` / `limbofilter` on a Velocity 3.5 server.
- Server starts with **no** `unsupported version: 26.2` and **no** `ProxyInitializeEvent` failure
  from limbofilter.
- Connect with a **26.2 (protocol 776)** client → captcha renders → land in limbo →
  `/register` + `/login` work → forwarded to lobby.
- Also smoke-test a 26.1 client to confirm no regression.

---

## 7. LimboAuth status (for the auth side of the handoff)

- LimboAuth has **no** protocol/packet/version code — it is bcrypt/TOTP/ORMLite auth logic. It is
  **not** involved in this bug and needs **no** change for it.
- LimboAuth was already rebuilt for Velocity 3.5 / MC 26.2:
  `velocityVersion 3.4.0-SNAPSHOT → 3.5.0-SNAPSHOT`, `limboapiVersion → 1.1.27`. Builds green,
  loads on the proxy.
- Unrelated runtime gotcha already seen & resolved on the test server: a MySQL
  `Access denied for user 'limboauth'@'localhost'` — that was a DB credential/config issue in
  `plugins/limboauth/config.yml`, **not** a code bug. Keep `storage-type: MYSQL` (H2 won't survive
  a bot flood) and ensure the configured user/password/database match the MySQL grant.

---

## 8. Constraints

- Work on the **fork**, on a feature branch; don't touch upstream `Elytrium/*`.
- Keep the diff minimal: the generator enum + `gameVersion` (Option A), or the fallback shims
  (Option B). Don't refactor unrelated protocol code.
- Don't change packet IDs or the already-correct `>= MINECRAFT_26_1` comparisons.
- List every changed file in the commit message.
