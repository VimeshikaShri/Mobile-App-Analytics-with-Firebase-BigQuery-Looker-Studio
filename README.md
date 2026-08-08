# AuraBuy: Mobile App Analytics with Firebase, BigQuery & Looker Studio

A React Native e-commerce demo app instrumented end-to-end with Firebase Analytics (GA4), exported to BigQuery, queried with SQL, and visualized in a live Looker Studio dashboard.

This project was built to demonstrate the full mobile analytics pipeline a real product team would use, not just "add a tracking library," but the actual chain: **instrument → validate → export → query → visualize**.

![](https://github.com/VimeshikaShri/Mobile-App-Analytics-with-Firebase-BigQuery-Looker-Studio/blob/main/4.png)

**Live dashboard:** [View on Looker Studio](https://datastudio.google.com/s/qCoew2ajoww)

**Google Data Studio:** [View full GA4 events report (PDF)](Mobile_App.pdf)

---

## Table of Contents

- [What This Project Does](#what-this-project-does)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [The Mobile App](#the-mobile-app)
- [Analytics Instrumentation](#analytics-instrumentation)
- [Validating Events (DebugView)](#validating-events-debugview)
- [Results: What the Data Shows](#results-what-the-data-shows)
- [BigQuery Export & SQL Analysis](#bigquery-export--sql-analysis)
- [Dashboard](#dashboard)
- [Key Technical Challenges & How They Were Solved](#key-technical-challenges--how-they-were-solved)
- [Running This Project Locally](#running-this-project-locally)
- [Project Structure](#project-structure)
- [What I'd Do Next](#what-id-do-next)

---

## What This Project Does

AuraBuy is a small e-commerce demo app (login → browse products → cart → checkout → purchase) built specifically to have a realistic conversion funnel to track. The point of the project is not the app itself, it is everything wired up *around* it:

1. A React Native app fires structured, parameterized analytics events at each step of the user journey.
2. Firebase's **native** Analytics SDK captures those events on-device and sends them to a GA4 property.
3. GA4 exports the raw event data to **BigQuery** on a daily schedule.
4. **SQL** queries against the raw `events_*` tables answer questions GA4's UI can't (custom aggregations, funnel drop-off, etc.).
5. A **Looker Studio** dashboard, connected live to BigQuery, presents the results.

This mirrors how analytics actually works at companies with a real data stack, GA4 is the collection layer, not the analysis layer while BigQuery and SQL are where the real questions get answered.

---

## Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────┐      ┌───────────┐
│  React Native    │      │  Firebase         │      │  BigQuery   │      │  Looker   │
│  App (Expo)      │ ───► │  Analytics (GA4)  │ ───► │  (raw event │ ───► │  Studio   │
│  native SDK      │      │  DebugView +      │      │  export)    │      │  dashboard│
│  event calls     │      │  production data  │      │  + SQL      │      │           │
└─────────────────┘      └──────────────────┘      └─────────────┘      └───────────┘
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Mobile framework | React Native (Expo, Expo Router, TypeScript) |
| Native build | Expo Dev Client (custom development build, not Expo Go) |
| Analytics SDK | `@react-native-firebase/app` + `@react-native-firebase/analytics` (native, modular API) |
| Backend/config | Firebase (GA4 property, `google-services.json`) |
| Native tooling | Android Studio, Android SDK, Gradle |
| Data warehouse | Google BigQuery (linked export from GA4) |
| Query language | SQL (standard BigQuery dialect) |
| Visualization | Looker Studio (formerly Data Studio), connected live to BigQuery |
| Version control | Git / GitHub |

---

## The Mobile App

The app is intentionally minimal, four screens, enough to generate every event in the funnel, nothing more:

| Screen | Purpose | Events fired |
|---|---|---|
| **Login / Sign Up** | Email input, "Continue" button | `sign_up`, `login` |
| **Products** | Search bar + product list with "Add to cart" | `search`, `add_to_cart` |
| **Cart** | Line items + "Checkout" button | `begin_checkout` |
| **Checkout** | Order total + "Confirm Purchase" button | `purchase` |

State is managed with a single `useState` string (`'login'` → `'products'` → `'cart'` → `'checkout'` → `'done'`) rather than a navigation library. This was a deliberate choice: with only four linear screens, `react-navigation` adds setup overhead and dependency weight without adding capability — a `switch`-style conditional render is simpler, easier to read, and does the same job.

The app was scaffolded with `npx create-expo-app`, which currently defaults to **Expo Router** (file-based routing). Because of this, the actual entry point lived at `src/app/index.tsx`, not `App.js`, a template detail worth knowing since older React Native tutorials assume the latter.

### `src/app/index.tsx`

```tsx
import { getAnalytics, logAddToCart, logBeginCheckout, logLogin, logPurchase, logSearch, logSignUp } from '@react-native-firebase/analytics';
import { useState } from 'react';
import { Button, FlatList, StyleSheet, Text, TextInput, View } from 'react-native';

const PRODUCTS = [
  { id: '1', name: 'T-Shirt', price: 15 },
  { id: '2', name: 'Sneakers', price: 60 },
  { id: '3', name: 'Backpack', price: 35 },
];

export default function Index() {
  const [screen, setScreen] = useState('login');
  const [email, setEmail] = useState('');
  const [cart, setCart] = useState([]);
  const [search, setSearch] = useState('');

  const filteredProducts = PRODUCTS.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase())
  );

  if (screen === 'login') {
    return (
      <View style={styles.box}>
        <Text style={styles.title}>Sign Up / Login</Text>
        <TextInput style={styles.input} placeholder="Email" value={email} onChangeText={setEmail} />
        <Button title="Continue" onPress={async () => {
          await logSignUp(getAnalytics(), { method: 'email' });
          await logLogin(getAnalytics(), { method: 'email' });
          setScreen('products');
        }} />
      </View>
    );
  }

  if (screen === 'products') {
    return (
      <View style={styles.box}>
        <Text style={styles.title}>Products</Text>
        <TextInput
          style={styles.input}
          placeholder="Search products"
          value={search}
          onChangeText={(text) => {
            setSearch(text);
            if (text.length > 2) logSearch(getAnalytics(), { search_term: text });
          }}
        />
        <FlatList
          data={filteredProducts}
          keyExtractor={item => item.id}
          renderItem={({ item }) => (
            <View style={styles.row}>
              <Text>{item.name} — ${item.price}</Text>
              <Button title="Add to cart" onPress={async () => {
                await logAddToCart(getAnalytics(), { item_id: item.id, item_name: item.name, value: item.price, currency: 'USD' });
                setCart([...cart, item]);
              }} />
            </View>
          )}
        />
        <Button title={`Go to Cart (${cart.length})`} onPress={() => setScreen('cart')} />
      </View>
    );
  }

  if (screen === 'cart') {
    return (
      <View style={styles.box}>
        <Text style={styles.title}>Cart</Text>
        {cart.map((item, i) => <Text key={i}>{item.name} — ${item.price}</Text>)}
        <Button title="Checkout" onPress={async () => {
          const total = cart.reduce((sum, i) => sum + i.price, 0);
          await logBeginCheckout(getAnalytics(), { value: total, currency: 'USD' });
          setScreen('checkout');
        }} />
      </View>
    );
  }

  if (screen === 'checkout') {
    return (
      <View style={styles.box}>
        <Text style={styles.title}>Checkout</Text>
        <Text>Total: ${cart.reduce((sum, i) => sum + i.price, 0)}</Text>
        <Button title="Confirm Purchase" onPress={async () => {
          const total = cart.reduce((sum, i) => sum + i.price, 0);
          await logPurchase(getAnalytics(), { value: total, currency: 'USD' });
          setScreen('done');
        }} />
      </View>
    );
  }

  return (
    <View style={styles.box}>
      <Text style={styles.title}>Thank you for your purchase!</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  box: { flex: 1, justifyContent: 'center', padding: 20, marginTop: 40 },
  title: { fontSize: 22, fontWeight: 'bold', marginBottom: 20 },
  input: { borderWidth: 1, borderColor: '#ccc', padding: 10, marginBottom: 10, borderRadius: 6 },
  row: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginBottom: 10 },
});
```

**Notable implementation detail — the modular Analytics API:** `@react-native-firebase/analytics` v22+ deprecated the old namespaced call style (`analytics().logEvent(...)`). This project uses v26, which requires the **modular API**, importing individual functions (`logPurchase`, `logAddToCart`, etc.) and passing a `getAnalytics()` instance as the first argument to each. Getting this right required checking the installed package version rather than assuming older tutorials' syntax would work.

---

## Analytics Instrumentation

| Event | Fired when | Parameters logged |
|---|---|---|
| `sign_up` | User taps "Continue" on login | `method: 'email'` |
| `login` | Same tap (demo treats sign-up and login as one step) | `method: 'email'` |
| `search` | User types 3+ characters in the product search box | `search_term` |
| `add_to_cart` | User taps "Add to cart" on a product | `item_id`, `item_name`, `value`, `currency` |
| `begin_checkout` | User taps "Checkout" from the cart | `value` (cart total), `currency` |
| `purchase` | User taps "Confirm Purchase" | `value` (order total), `currency` |
| `app_open` | Automatic — fired by the Firebase SDK on launch | *(no manual code needed)* |

`currency` is required by GA4 on any event that includes a `value`: Omitting it throws a runtime validation error (`if you supply the 'value' parameter, you must also supply the 'currency' parameter`), which is how this requirement was actually discovered during development.

---

## Validating Events (DebugView)

Before trusting any dashboard, every event above was verified live in **Firebase DebugView**: GA4's real-time event stream for a single test device. This was done by:

```bash
adb shell setprop debug.firebase.analytics.app com.vimeshika.myshopapp
```

...which flags the app for instant (rather than batched/hourly) event delivery, then walking through the full app flow while watching events land in the Firebase console within seconds — confirming both that events fired *and* that their parameters were structured correctly, before ever writing a line of SQL against them.

---

## Results: What the Data Shows

Pulled from the GA4 **Events** report (Firebase console → Analytics), reflecting live usage during development and testing:

[Firebase_overview (PDF)](https://github.com/VimeshikaShri/Mobile-App-Analytics-with-Firebase-BigQuery-Looker-Studio/blob/main/Firebase_overview.pdf)

| Event name | Event count | % of total | Total users |
|---|---:|---:|---:|
| `screen_view` | 157 | 62.8% | 4 |
| `user_engagement` | 61 | 24.4% | 4 |
| `session_start` | 9 | 3.6% | 4 |
| `first_open` | 4 | 1.6% | 4 |
| `login` | 4 | 1.6% | 2 |
| `sign_up` | 4 | 1.6% | 2 |
| `app_remove` | 3 | 1.2% | 3 |
| `begin_checkout` | 3 | 1.2% | 1 |
| `purchase` | 3 | 1.2% | 1 |
| `app_clear_data` | 2 | 0.8% | 1 |
| **Total** | **250** | **100%** | **4** |

**Device & geography** (from Firebase's Active Users breakdowns): 100% Android, 100% mobile category, device model Samsung SM-S731B, located in Chennai, India, consistent with this being a controlled test deployment on a single physical device rather than a public release, which is expected and disclosed here rather than presented as organic traffic.

**Funnel snapshot:** of the events tracked, `begin_checkout` → `purchase` shows no drop-off in this dataset (3 → 3), since testing sessions were run end-to-end rather than abandoned mid-funnel — a realistic funnel with organic traffic would be expected to show attrition at this step.

---

## BigQuery Export & SQL Analysis

GA4 was linked to BigQuery via **Firebase Console → Project Settings → Integrations → BigQuery**, enabling daily export of raw, unsampled event data into a dataset named `analytics_<GA4_PROPERTY_ID>` — in this project, `analytics_548512384` inside project `my-shop-demo-da9be`.

![BigQuery (Google Cloud Console](https://github.com/VimeshikaShri/Mobile-App-Analytics-with-Firebase-BigQuery-Looker-Studio/blob/main/6.png)

Each day of events lands in its own table, `events_YYYYMMDD`. Querying across all of them uses BigQuery's wildcard table syntax (`events_*`) rather than naming individual dates.

### Query: Event counts, most to least frequent

```sql
SELECT event_name, COUNT(*) as event_count
FROM `my-shop-demo-da9be.analytics_548512384.events_*`
GROUP BY event_name
ORDER BY event_count DESC
```

**What this does, line by line:**
- `FROM ...events_*`: Scans every daily events table in the dataset as if it were one combined table.
- `COUNT(*)`: Counts every row (i.e., every individual event instance) per group.
- `GROUP BY event_name`: Collapses all rows into one row per distinct event type.
- `ORDER BY event_count DESC`: Ranks the most frequent events first.

This is functionally the SQL equivalent of the "Results" table above, the advantage of running it directly in BigQuery instead of just reading the GA4 UI is that this query can be extended (date filtering, joining against user properties, computing custom funnel math) in ways the GA4 interface doesn't expose.

### Additional queries this schema supports (not yet run, included as extension ideas)

**Checkout-to-purchase conversion rate:**
```sql
SELECT
  COUNTIF(event_name = 'begin_checkout') AS checkouts_started,
  COUNTIF(event_name = 'purchase') AS purchases_completed,
  ROUND(COUNTIF(event_name = 'purchase') / NULLIF(COUNTIF(event_name = 'begin_checkout'), 0) * 100, 1) AS conversion_pct
FROM `my-shop-demo-da9be.analytics_548512384.events_*`
```

**Average order value from purchase events:**
```sql
SELECT
  AVG((SELECT value.double_value FROM UNNEST(event_params) WHERE key = 'value')) AS avg_order_value
FROM `my-shop-demo-da9be.analytics_548512384.events_*`
WHERE event_name = 'purchase'
```
*(Note: `event_params` is a repeated/nested field in the raw GA4 export schema, extracting a specific parameter requires `UNNEST`, unlike the flat `event_name` column used in the main query above.)*

---

## Dashboard

Built in Looker Studio, connected as a **live data source to BigQuery** (not a static export), the dashboard reflects new data automatically as it lands.

**[→ View the live dashboard](https://datastudio.google.com/s/qCoew2ajoww)**

Includes:
- KPI scorecards (sessions, engagement rate, event count, checkouts, WAU/MAU)
- Ranked bar chart of top events by volume
- Session engagement trends over time
- Device and geography breakdowns

---

## Key Technical Challenges & How They Were Solved

Documenting these because they reflect real debugging, not a clean tutorial run:

| Problem | Root Cause | Fix |
|---|---|---|
| App did not open after adding Firebase | Plain **Expo Go** cannot run native modules (`@react-native-firebase`), it requires a **custom development build** | Switched from `npx expo start` to `npx expo run:android`, which compiles and installs a dev-client build |
| `google-services.json` errors during build | `expo prebuild` crashed partway through on an unrelated **iOS** config step (missing `GoogleService-Info.plist`), aborting before it reached the Android-specific Google Services Gradle plugin setup | Ran `expo prebuild --clean --platform android` to skip iOS generation entirely |
| `analytics().logSignUp is not a function` | `@react-native-firebase/analytics` v26 removed the old namespaced API in favor of a **modular** one | Rewrote all event calls to the `logX(getAnalytics(), {...})` pattern |
| `No Firebase App '[DEFAULT]' has been created` | Native Firebase init never completed because the Google Services Gradle plugin hadn't been applied yet (see prebuild issue above) | Same fix — full clean rebuild after correcting the prebuild command |
| `if you supply 'value', you must supply 'currency'` | GA4 requires `currency` alongside any `value` parameter | Added `currency: 'USD'` to every event with a monetary value |
| App stuck on "Thank you for your purchase" regardless of restarts, reinstalls, or network fixes | A copy-paste accident had corrupted the initial state string, `useState('login')` had become `useState('logawait logAddToCart(...)in')`, so `screen === 'login'` was never true | Found via `cat`-ing the file directly instead of continuing to debug the network/build layer |
| Metro connection failures ("Host unreachable" / "Failed to connect") | USB tunnel (`adb reverse`) and Metro's dev server state didn't reliably survive multiple restarts during a long debugging session | Standardized on explicitly re-running `adb reverse tcp:8081 tcp:8081` immediately before every reconnect attempt |

---

## Running This Project Locally

```bash
# 1. Install dependencies
npm install

# 2. Add your own Firebase config
#    Place google-services.json in the project root
#    Update app.json's android.package to match your Firebase app registration

# 3. Build and install (requires Android Studio + SDK installed)
npx expo prebuild --clean --platform android
npx expo run:android

# 4. Start the dev server for subsequent code changes
npx expo start --dev-client
```

**Requirements:** Node.js, Android Studio (SDK + emulator or a physical Android device), a Firebase project with Analytics enabled.

---

## Project Structure

```
my-shop-app/
├── src/
│   └── app/
│       └── index.tsx          # Entry point — all screens, all analytics event calls
├── google-services.json       # Firebase config (not committed — see .gitignore)
├── app.json                   # Expo config: package name, plugins, Firebase file reference
├── package.json
└── android/                   # Auto-generated native project (via expo prebuild)
```

---

## **Contact**

**<small>Vimeshika Shri : GitHub: [@VimeshikaShri](https://github.com/VimeshikaShri)</small>**
