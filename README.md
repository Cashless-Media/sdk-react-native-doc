# MyCashless React Native SDK

SDK oficial de MyCashless para React Native. Integra funcionalidad de pagos cashless, tickets, wallet y sincronización en aplicaciones React Native y Expo.

## Tabla de Contenidos

- [Instalación](#instalación)
- [Guía Rápida de Integración](#guía-rápida-de-integración)
- [Uso Básico](#uso-básico)
- [Flujos Completos](#flujos-completos)
- [Adaptadores de Almacenamiento](#adaptadores-de-almacenamiento)
- [API Reference](#api-reference)
- [React Hooks](#react-hooks)
- [Servicios](#servicios)
- [Tipos de Transacción](#tipos-de-transacción)
- [Constantes](#constantes)
- [Utilidades](#utilidades)
- [Logger](#logger)
- [Testing](#testing)
- [Seguridad](#seguridad)
- [Soporte](#soporte)

## Instalación

```bash
npm install @mycashless/react-native-sdk
# o
yarn add @mycashless/react-native-sdk
```

### Instalación desde archivo .tgz

Si recibiste el SDK como archivo `.tgz`, instálalo directamente:

```bash
npm install ./mycashless-react-native-sdk-1.0.0.tgz
```

### Dependencias Requeridas

```bash
# Storage (key-value persistente)
npm install @react-native-async-storage/async-storage

# SQLite (base de datos local)
npm install @op-engineering/op-sqlite

# Para pagos con Stripe (opcional, solo si usas recargas con tarjeta)
npm install @stripe/stripe-react-native
```

### Alternativas de Dependencias

Si prefieres otras librerías de storage/database, el SDK acepta cualquier implementación que cumpla con las interfaces `StorageAdapter` y `DatabaseAdapter` (ver [Adaptadores de Almacenamiento](#adaptadores-de-almacenamiento)):

```bash
# Alternativa storage: react-native-keychain o expo-secure-store
npm install react-native-keychain
# o
npm install expo-secure-store

# Alternativa SQLite: react-native-sqlite-storage o expo-sqlite
npm install react-native-sqlite-storage
# o
npm install expo-sqlite
```

## Guía Rápida de Integración

Esta sección cubre todo lo necesario para integrar el SDK en tu app desde cero. Si solo recibiste el archivo `.tgz` y esta documentación, sigue estos pasos.

### Paso 1: Crear los Adaptadores de Almacenamiento

El SDK necesita dos adaptadores que tú provees: uno para key-value storage y otro para SQLite. Crea el archivo `src/adapters/StorageAdapters.ts`:

```typescript
import AsyncStorage from '@react-native-async-storage/async-storage';
import { open, type DB } from '@op-engineering/op-sqlite';
import type {
  StorageAdapter,
  DatabaseAdapter,
  DatabaseResult,
  DatabaseTransaction,
} from '@mycashless/react-native-sdk';

// ============================================
// AsyncStorage Adapter (key-value persistente)
// ============================================
export class AsyncStorageAdapter implements StorageAdapter {
  async getItem(key: string): Promise<string | null> {
    return AsyncStorage.getItem(key);
  }

  async setItem(key: string, value: string): Promise<void> {
    await AsyncStorage.setItem(key, typeof value === 'string' ? value : String(value));
  }

  async removeItem(key: string): Promise<void> {
    await AsyncStorage.removeItem(key);
  }

  async clear(): Promise<void> {
    await AsyncStorage.clear();
  }
}

// ============================================
// op-sqlite Database Adapter (SQLite)
// ============================================

function sanitizeParams(params?: unknown[]): any[] | undefined {
  if (!params) return undefined;
  return params.map(p => {
    if (p === undefined || p === null) return null;
    if (typeof p === 'boolean') return p ? 1 : 0;
    if (typeof p === 'object') return JSON.stringify(p);
    return p;
  });
}

function buildResult(raw: any): DatabaseResult {
  const rows = raw.rows ?? [];
  return {
    rows: {
      length: rows.length,
      item(index: number) { return rows[index]; },
      raw() { return rows; },
    },
    insertId: raw.insertId,
    rowsAffected: raw.rowsAffected ?? 0,
  };
}

export class OpSqliteDatabaseAdapter implements DatabaseAdapter {
  private db: DB | null = null;
  private dbName: string;

  constructor(dbName: string = 'mycashless.db') {
    this.dbName = dbName;
  }

  private getDB(): DB {
    if (!this.db) {
      this.db = open({ name: this.dbName });
    }
    return this.db;
  }

  async executeSql(sql: string, params?: unknown[]): Promise<DatabaseResult> {
    const db = this.getDB();
    const result = await db.execute(sql, sanitizeParams(params) as any[]);
    return buildResult(result);
  }

  async transaction(callback: (tx: DatabaseTransaction) => void): Promise<void> {
    const db = this.getDB();
    await db.execute('BEGIN TRANSACTION');
    try {
      const tx: DatabaseTransaction = {
        executeSql: async (sql: string, params?: unknown[]): Promise<DatabaseResult> => {
          const result = await db.execute(sql, sanitizeParams(params) as any[]);
          return buildResult(result);
        },
      };
      callback(tx);
      await db.execute('COMMIT');
    } catch (error) {
      await db.execute('ROLLBACK');
      throw error;
    }
  }

  async close(): Promise<void> {
    if (this.db) {
      this.db.close();
      this.db = null;
    }
  }
}
```

### Paso 2: Inicializar el SDK en App.tsx

Inicializa el SDK una sola vez al arrancar la app. Esto se hace en el componente raíz (`App.tsx`):

```tsx
import React, { useEffect, useState } from 'react';
import { View, ActivityIndicator, Text, StyleSheet } from 'react-native';
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import { AsyncStorageAdapter, OpSqliteDatabaseAdapter } from './src/adapters/StorageAdapters';

function App(): React.JSX.Element {
  const [sdkReady, setSdkReady] = useState(false);
  const [initError, setInitError] = useState<string | null>(null);

  useEffect(() => {
    const initSDK = async () => {
      try {
        const sdk = MyCashlessSDK.getInstance();
        if (!sdk.isInitialized()) {
          await sdk.initialize(
            {
              production: true,   // true = producción, false = staging
              debug: __DEV__,     // logs en consola solo en desarrollo
            },
            {
              storage: new AsyncStorageAdapter(),
              database: new OpSqliteDatabaseAdapter(),
            },
          );
        }
        setSdkReady(true);
      } catch (error: any) {
        console.error('SDK init failed:', error);
        setInitError(error.message || 'SDK initialization failed');
        setSdkReady(true); // Permite navegación para debugging
      }
    };
    initSDK();
  }, []);

  if (!sdkReady) {
    return (
      <View style={styles.splash}>
        <ActivityIndicator size="large" />
        <Text style={styles.splashText}>Initializing...</Text>
      </View>
    );
  }

  return (
    // Tu NavigationContainer y Stack aquí
  );
}

const styles = StyleSheet.create({
  splash: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  splashText: { marginTop: 16, fontSize: 16 },
});
```

### Paso 3: Entorno de Producción vs Desarrollo

La configuración del SDK se controla con dos flags:

| Flag | Producción | Desarrollo/Staging |
|------|-----------|-------------------|
| `production` | `true` | `false` |
| `debug` | `false` (o `__DEV__`) | `true` |

- **`production: true`** — Apunta a `https://api.mycashless.com` (API de producción)
- **`production: false`** — Apunta a `https://staging-api.mycashless.com` (API de staging/desarrollo)
- **`debug`** — Solo activa logs en consola. No afecta el entorno de la API. Puedes usar `__DEV__` para que se active automáticamente solo en builds de desarrollo.

```typescript
// Producción
await sdk.initialize({ production: true, debug: false }, { storage, database });

// Desarrollo / Staging
await sdk.initialize({ production: false, debug: true }, { storage, database });
```

### Paso 4: Usar el SDK

Una vez inicializado, el SDK restaura automáticamente la sesión previa (evento, wallet, login) si existe. El flujo típico después de la inicialización:

```typescript
const sdk = MyCashlessSDK.getInstance();

// 1. Conectar a un evento (solo la primera vez o al cambiar de evento)
const event = await sdk.connectByCode('CODIGO-EVENTO');

// 2. Login del usuario
const result = await sdk.authService.login('+521234567890', 'password');

// 3. Sincronizar conexión (obtiene balance del servidor, dchip_id, config)
await sdk.syncEventConnection();

// 4. Iniciar sincronización periódica (cada 5 min, en background)
sdk.startSync();

// 5. Leer balance
const balance = sdk.walletService.getBalance();
// balance.balance = 5000 (centavos → $50.00)
// balance.promo = 1000 (centavos → $10.00)
```

En las siguientes aperturas de la app, `sdk.initialize()` restaura todo automáticamente — no es necesario reconectar al evento ni re-hacer login.

---

## Uso Básico

### 1. Inicialización

```typescript
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import { KeychainAdapter, SQLiteAdapter } from './adapters'; // tus adaptadores

async function initializeApp() {
  const sdk = MyCashlessSDK.getInstance();

  await sdk.initialize(
    {
      production: true,       // false para staging
      language: 'es',
      debug: __DEV__,
    },
    {
      storage: new KeychainAdapter(),
      database: new SQLiteAdapter(),
    }
  );
}
```

### 2. Autenticación

```typescript
import { useAuth } from '@mycashless/react-native-sdk';

function LoginScreen() {
  const { login, isLoading, error, user } = useAuth();

  const handleLogin = async () => {
    const result = await login('+521234567890', 'password123');
    if (result.success) {
      // Usuario autenticado
    }
  };

  return (
    <View>
      {isLoading && <ActivityIndicator />}
      {error && <Text>{error}</Text>}
      <Button title="Login" onPress={handleLogin} />
    </View>
  );
}
```

### 3. Conectar a un Evento

```typescript
const sdk = MyCashlessSDK.getInstance();

// Por código del evento
const event = await sdk.connectByCode('mi-evento');

// O por UID directo
const event = await sdk.setEvent('event-uid-123');

// Iniciar sincronización periódica
sdk.startSync();
```

### 4. Wallet y Balance

```typescript
import { useWallet } from '@mycashless/react-native-sdk';

function WalletScreen() {
  const {
    balance,           // balance en centavos
    promo,             // promo en centavos
    total,             // balance + promo
    tokens,            // tokens disponibles
    balanceFormatted,  // "$100.00"
    promoFormatted,    // "$25.00"
    totalFormatted,    // "$125.00"
    refreshWallet,
    getTransactionHistory,
    hasSufficientBalance,
    isLoading,
  } = useWallet('MXN');

  return (
    <View>
      <Text>Balance: {balanceFormatted}</Text>
      <Text>Promo: {promoFormatted}</Text>
      <Text>Total: {totalFormatted}</Text>
      <Button title="Actualizar" onPress={refreshWallet} disabled={isLoading} />
    </View>
  );
}
```

### 5. Recargas con Stripe

```typescript
import { MyCashlessSDK, PaymentService } from '@mycashless/react-native-sdk';
import { initStripe, usePaymentSheet } from '@stripe/stripe-react-native';

function ReloadScreen() {
  const { initPaymentSheet, presentPaymentSheet } = usePaymentSheet();
  const sdk = MyCashlessSDK.getInstance();
  const paymentService = sdk.paymentService;

  const handleReload = async () => {
    // 1. Obtener configuración de Stripe
    const stripeKey = await paymentService.getStripePublishableKey();
    const connectedAccountId = await paymentService.getConnectedAccountId();

    await initStripe({
      publishableKey: stripeKey,
      stripeAccountId: connectedAccountId ?? undefined,
    });

    // 2. Obtener montos de recarga disponibles
    const amounts = await paymentService.getReloadAmounts();
    const selected = amounts[0]; // ejemplo: { id: 1, amount: 10000, label: "$100" }

    // 3. Crear PaymentIntent
    const reloadDetail = paymentService.buildReloadDetail([
      { id: selected.id, quantity: 1 }
    ]);
    const intent = await paymentService.createPaymentIntent(reloadDetail);
    if (!intent) return;

    // 4. Configurar PaymentSheet
    const { error: initError } = await initPaymentSheet({
      paymentIntentClientSecret: intent.client_secret,
      merchantDisplayName: 'MyCashless',
      stripeAccountId: connectedAccountId ?? undefined,
    });
    if (initError) return;

    // 5. Presentar PaymentSheet
    const { error: paymentError } = await presentPaymentSheet();
    if (paymentError) return;

    // 6. Pago exitoso — sincronizar transfers del backend
    const result = await sdk.syncNow();
    if (result.transfersSynced > 0) {
      Alert.alert('Recarga exitosa', 'Tu balance ha sido actualizado.');
    } else {
      // Fallback: aplicar recarga localmente si el backend tarda
      await sdk.applyReloadLocally(selected.amount);
      Alert.alert('Recarga exitosa', 'Tu balance ha sido actualizado.');
    }
  };
}
```

### 6. Procesamiento de QR

```typescript
const sdk = MyCashlessSDK.getInstance();

// Procesar un QR escaneado
const result = await sdk.processScannedQR(encryptedQRData);

if (result.status === QRCodeStatus.VALID) {
  console.log('Transacción exitosa:', result.transaction);
  console.log('Nuevo balance:', result.newBalance);
  // La sincronización al backend se dispara automáticamente
}
```

### 7. Sincronización

```typescript
import { useSync } from '@mycashless/react-native-sdk';

function SyncStatus() {
  const {
    isSyncing,
    lastSyncTime,
    lastSyncResult,
    syncNow,
    startAutoSync,
    stopAutoSync,
    error,
  } = useSync();

  return (
    <View>
      {isSyncing && <Text>Sincronizando...</Text>}
      {lastSyncTime && (
        <Text>Última sync: {lastSyncTime.toLocaleString()}</Text>
      )}
      {lastSyncResult && (
        <Text>
          Transacciones: {lastSyncResult.transactionsSynced},
          Transfers: {lastSyncResult.transfersSynced}
        </Text>
      )}
      <Button title="Sincronizar" onPress={syncNow} />
    </View>
  );
}
```

## Flujos Completos

Los siguientes flujos documentan la secuencia exacta de métodos del SDK necesarios para cada operación, tal como se implementan en las apps nativas de iOS y Android.

### Flujo 1: Inicialización + Conexión a Evento + Login

Este es el flujo principal que toda app debe seguir al arrancar.

```typescript
import {
  MyCashlessSDK,
  CachedData,
  StorageKeys,
  STRIPE_KEYS,
} from '@mycashless/react-native-sdk';

async function fullSetup() {
  const sdk = MyCashlessSDK.getInstance();

  // ── Paso 1: Inicializar SDK ──
  await sdk.initialize(
    { production: true, language: 'es', debug: __DEV__ },
    { storage: new MyStorageAdapter(), database: new MyDatabaseAdapter() }
  );

  // ── Paso 2: Conectar a evento por código ──
  const event = await sdk.connectByCode('mi-evento');
  // O por UID: const event = await sdk.setEvent('event-uid');
  if (!event) throw new Error('Evento no encontrado');

  // ── Paso 3: Verificar si el teléfono existe ──
  const authService = sdk.authService;
  const phoneExists = await authService.checkPhoneExists('+521234567890');

  if (phoneExists) {
    // ── Paso 4a: Login ──
    const result = await authService.login('+521234567890', 'password');
    if (!result.success) throw new Error(result.error);
  } else {
    // ── Paso 4b: Registro ──
    const result = await authService.register({
      phone: '+521234567890',
      password: 'password',
      first_name: 'Juan',
      last_name: 'Pérez',
      email: 'juan@example.com',
    });
    if (!result.success) throw new Error(result.error);
  }

  // ── Paso 5: Sincronizar conexión del usuario al evento ──
  // Obtiene dchip_id, balance del servidor, configuración del evento
  const connection = await sdk.syncEventConnection();

  // ── Paso 6: Iniciar sincronización periódica (cada 5 min) ──
  sdk.startSync();

  // ── Listo: balance, promo y tokens están disponibles ──
  const balance = sdk.walletService.getBalance();
  console.log('Balance:', balance.balance / 100);
  console.log('Promo:', balance.promo / 100);
}
```

**Métodos involucrados:**

| Paso | Método | Servicio |
|------|--------|----------|
| 1 | `sdk.initialize(config, adapters)` | MyCashlessSDK |
| 2 | `sdk.connectByCode(code)` o `sdk.setEvent(uid)` | MyCashlessSDK |
| 3 | `authService.checkPhoneExists(phone)` | AuthService |
| 4a | `authService.login(phone, password)` | AuthService |
| 4b | `authService.register(userData)` | AuthService |
| 5 | `sdk.syncEventConnection()` | MyCashlessSDK |
| 6 | `sdk.startSync()` | MyCashlessSDK |

### Flujo 1b: Login con Fanki Token (SSO)

Para integraciones con Fanki, el SDK soporta autenticación transparente usando el Bearer token del usuario ya autenticado en Fanki. El backend valida el token contra la API de Fanki, crea el usuario en MyCashless si no existe (con password aleatorio), y devuelve un JWT de MyCashless.

```typescript
import { MyCashlessSDK } from '@mycashless/react-native-sdk';

async function fankiSetup(fankiToken: string) {
  const sdk = MyCashlessSDK.getInstance();

  // ── Paso 1: Inicializar SDK ──
  await sdk.initialize(
    { production: true, language: 'es', debug: __DEV__ },
    { storage: new MyStorageAdapter(), database: new MyDatabaseAdapter() }
  );

  // ── Paso 2: Login con token de Fanki (sin password, sin UI de login) ──
  const result = await sdk.authService.loginWithFankiToken(fankiToken);
  if (!result.success) throw new Error(result.error);

  // ── Paso 3: Obtener eventos activos de Fanki ──
  const events = await sdk.authService.getFankiEvents();
  // → [{ id, uid, name, nick_name, date_start, date_end }]

  // ── Paso 4: Conectar al evento (por uid o nickname) ──
  const event = await sdk.setEvent(events[0].uid);
  if (!event) throw new Error('Evento no encontrado');

  // ── Paso 5: Sincronizar conexión del usuario al evento ──
  const connection = await sdk.syncEventConnection();

  // ── Paso 6: Iniciar sincronización periódica ──
  sdk.startSync();

  // ── Listo ──
  const balance = sdk.walletService.getBalance();
  console.log('Balance:', balance.balance / 100);
}
```

**Métodos involucrados:**

| Paso | Método | Servicio |
|------|--------|----------|
| 1 | `sdk.initialize(config, adapters)` | MyCashlessSDK |
| 2 | `authService.loginWithFankiToken(fankiToken)` | AuthService |
| 3 | `authService.getFankiEvents()` | AuthService |
| 4 | `sdk.setEvent(uid)` o `sdk.connectByCode(code)` | MyCashlessSDK |
| 5 | `sdk.syncEventConnection()` | MyCashlessSDK |
| 6 | `sdk.startSync()` | MyCashlessSDK |

> **Nota:** El `fankiToken` es el Bearer token que Fanki usa para autenticar al usuario en su propia API. El backend de MyCashless lo valida llamando a `GET /api/frontend-api/fan` con firma HMAC, extrae el teléfono del fan, y crea o encuentra al usuario correspondiente.

**Respuesta de `getFankiEvents()`:**

```typescript
interface FankiEvent {
  id: number;        // ID interno del evento
  uid: string;       // UID para usar con setEvent()
  name: string;      // Nombre del evento
  nick_name: string; // Código/nickname para usar con connectByCode()
  date_start: string; // Fecha de inicio (ISO 8601)
  date_end: string;   // Fecha de fin (ISO 8601)
}
```

> El backend resuelve automáticamente los eventos asociados a Fanki — no necesitan conocer el `group_id` ni otros datos internos.

### Flujo 2: Consultar Balance y Transacciones

```typescript
const sdk = MyCashlessSDK.getInstance();
const walletService = sdk.walletService;

// ── Balance actual (en centavos) ──
const { balance, promo, tokens } = walletService.getBalance();

// ── Balance formateado (en unidades) ──
const formatted = walletService.getBalanceFormatted();
// formatted.balance = 100.00, formatted.promo = 25.00, formatted.total = 125.00

// ── Historial de transacciones ──
const transactions = await walletService.getTransactionHistory();
// Ordenadas por created_time DESC desde SQLite local

// ── Últimas N transacciones ──
const recent = await walletService.getRecentTransactions(10);

// ── Verificar si tiene saldo suficiente para un monto ──
const canPay = walletService.hasSufficientBalance(5000); // $50.00

// ── Obtener tokens disponibles ──
const tokenCounts = walletService.getTokens();
// { token1: 2, token2: 0, token3: 1, token4: 0, token5: 0 }
```

**Métodos involucrados:**

| Método | Servicio | Retorno |
|--------|----------|---------|
| `walletService.getBalance()` | WalletService | `WalletBalance` |
| `walletService.getBalanceFormatted()` | WalletService | `{ balance, promo, total, tokens }` |
| `walletService.getTransactionHistory()` | WalletService | `Transaction[]` |
| `walletService.getRecentTransactions(n)` | WalletService | `Transaction[]` |
| `walletService.hasSufficientBalance(amount)` | WalletService | `boolean` |
| `walletService.getTokens()` | WalletService | `TokenAmount` |
| `walletService.getTotalTokenCount()` | WalletService | `number` |

### Flujo 3: Recarga con Stripe (Tarjeta)

```typescript
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import { usePaymentSheet } from '@stripe/stripe-react-native';

const sdk = MyCashlessSDK.getInstance();
const paymentService = sdk.paymentService;
const { initPaymentSheet, presentPaymentSheet } = usePaymentSheet();

async function reloadWithStripe(selectedAmountId: number, quantity: number) {
  // ── Paso 1: Obtener configuración de Stripe ──
  const stripeKey = await paymentService.getStripePublishableKey();
  const connectedAccountId = await paymentService.getConnectedAccountId();

  // ── Paso 2: Obtener montos de recarga configurados en el evento ──
  const amounts = await paymentService.getReloadAmounts();

  // ── Paso 3: Calcular total con comisiones (si service_charge_mode > 0) ──
  const selected = amounts.find(a => a.id === selectedAmountId)!;
  const fees = await paymentService.calculateTotalWithFees(selected.amount * quantity);
  // fees = { subtotal: 10000, fee: 500, total: 10500, feePercent: 5 }

  // ── Paso 4: Construir detalle de recarga ──
  const reloadDetail = paymentService.buildReloadDetail([
    { id: selected.id, quantity }
  ]);

  // ── Paso 5: Crear PaymentIntent ──
  const intent = await paymentService.createPaymentIntent(reloadDetail);
  if (!intent) throw new Error('No se pudo crear el PaymentIntent');

  // Si el pago ya fue completado server-side (tarjeta guardada con auto-confirm)
  if (intent.alreadySucceeded) {
    await postPaymentSync();
    return;
  }

  // ── Paso 6: Configurar y presentar PaymentSheet ──
  await initPaymentSheet({
    paymentIntentClientSecret: intent.clientSecret,
    merchantDisplayName: 'MyCashless',
    stripeAccountId: connectedAccountId ?? undefined,
  });

  const { error } = await presentPaymentSheet();
  if (error) throw new Error(error.message);

  // ── Paso 7: Si requiere captura server-side ──
  if (intent.requiresCapture) {
    const piId = intent.clientSecret.split('_secret_')[0];
    await paymentService.confirmPaymentIntent(piId);
  }

  // ── Paso 8: Post-pago — sincronizar balance ──
  await postPaymentSync();
}

async function postPaymentSync() {
  // Intentar sync de transfers desde backend
  const result = await sdk.syncNow();

  if (result.transfersSynced > 0) {
    // Balance actualizado via transfers
    return;
  }

  // Fallback: sincronizar balance directo del servidor
  const updated = await sdk.syncBalanceFromServer();

  if (!updated) {
    // Último fallback: aplicar recarga localmente
    await sdk.applyReloadLocally(selectedAmount);
  }
}
```

**Métodos involucrados:**

| Paso | Método | Servicio |
|------|--------|----------|
| 1 | `paymentService.getStripePublishableKey()` | PaymentService |
| 1 | `paymentService.getConnectedAccountId()` | PaymentService |
| 2 | `paymentService.getReloadAmounts()` | PaymentService |
| 3 | `paymentService.calculateTotalWithFees(amount)` | PaymentService |
| 4 | `paymentService.buildReloadDetail(amounts)` | PaymentService |
| 5 | `paymentService.createPaymentIntent(detail)` | PaymentService |
| 6 | Stripe SDK (`initPaymentSheet`, `presentPaymentSheet`) | Stripe |
| 7 | `paymentService.confirmPaymentIntent(piId)` | PaymentService |
| 8a | `sdk.syncNow()` | MyCashlessSDK |
| 8b | `sdk.syncBalanceFromServer()` | MyCashlessSDK |
| 8c | `sdk.applyReloadLocally(amount)` | MyCashlessSDK |

### Flujo 4: Recarga con MercadoPago

```typescript
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import { Linking } from 'react-native';

const sdk = MyCashlessSDK.getInstance();
const paymentService = sdk.paymentService;

async function reloadWithMercadoPago(reloadDetail: string) {
  // ── Paso 1: Verificar disponibilidad ──
  const methods = await paymentService.getAvailablePaymentMethods();
  if (!methods.mercadoPago) throw new Error('MercadoPago no disponible');

  // ── Paso 2: Crear preferencia de pago ──
  const preference = await paymentService.createMercadoPagoPreference(reloadDetail);
  if (!preference) throw new Error('No se pudo crear la preferencia');

  // ── Paso 3: Abrir checkout de MercadoPago (browser/WebView) ──
  await Linking.openURL(preference.initPoint);
  // El usuario paga en MercadoPago y regresa a la app

  // ── Paso 4: Verificar pago (con reintentos exponenciales) ──
  const result = await paymentService.verifyMercadoPagoPayment(
    preference.externalReference
  );

  if (result.status === 'approved') {
    // ── Paso 5: Sincronizar balance ──
    await sdk.syncNow();
    // O fallback:
    await sdk.syncBalanceFromServer();
  } else if (result.status === 'pending') {
    // Pago en proceso
  } else {
    // Pago fallido
    const errorMsg = paymentService.getMercadoPagoErrorMessage(result.statusDetail);
    const suggestOther = paymentService.shouldSuggestDifferentMethod(result.statusDetail);
  }
}
```

**Métodos involucrados:**

| Paso | Método | Servicio |
|------|--------|----------|
| 1 | `paymentService.getAvailablePaymentMethods()` | PaymentService |
| 2 | `paymentService.createMercadoPagoPreference(detail)` | PaymentService |
| 3 | `Linking.openURL(preference.initPoint)` | React Native |
| 4 | `paymentService.verifyMercadoPagoPayment(ref)` | PaymentService |
| 4 | `paymentService.getMercadoPagoErrorMessage(detail)` | PaymentService |
| 5 | `sdk.syncNow()` / `sdk.syncBalanceFromServer()` | MyCashlessSDK |

### Flujo 5: Recarga con PayPal

```typescript
const sdk = MyCashlessSDK.getInstance();
const paymentService = sdk.paymentService;

async function reloadWithPayPal(reloadDetail: string) {
  // ── Paso 1: Obtener client ID ──
  const clientId = await paymentService.getPayPalClientId();

  // ── Paso 2: Crear orden ──
  const order = await paymentService.createPayPalOrder(reloadDetail);
  if (!order) throw new Error('No se pudo crear la orden');

  // ── Paso 3: Usuario aprueba la orden en PayPal UI ──
  // (integrar con @paypal/react-native-checkout o WebView)

  // ── Paso 4: Capturar la orden ──
  const capture = await paymentService.capturePayPalOrder(
    order.orderId,
    order.transferIds,
    order.tableIds,
    order.serviceFee,
  );

  // ── Paso 5: Sincronizar balance ──
  await sdk.syncNow();
}
```

### Flujo 6: Escaneo de QR (mPOS)

```typescript
import { MyCashlessSDK, QRCodeStatus } from '@mycashless/react-native-sdk';

const sdk = MyCashlessSDK.getInstance();

// El QR viene encriptado con AES desde el mPOS
async function handleQRScan(encryptedQRData: string) {
  const result = await sdk.processScannedQR(encryptedQRData);

  switch (result.status) {
    case QRCodeStatus.VALID:
      // Transacción aplicada exitosamente
      console.log('Tipo:', result.transactionType);  // BALANCE, SALE, TIP, etc.
      console.log('Monto:', result.amount);
      console.log('Nuevo balance:', result.newBalance);
      // syncNow() se ejecuta automáticamente en background
      break;

    case QRCodeStatus.INVALID_USED:
      // QR ya fue escaneado antes (duplicado)
      break;

    case QRCodeStatus.INVALID_BALANCE:
      // Balance insuficiente para la compra
      break;

    case QRCodeStatus.INVALID_ID:
      // QR no pertenece a este dispositivo
      break;

    case QRCodeStatus.INVALID_EVENT:
      // QR pertenece a otro evento
      break;

    case QRCodeStatus.INVALID_ENCRYPT:
      // No se pudo desencriptar (evento incorrecto o QR corrupto)
      break;
  }
}
```

**Métodos involucrados:**

| Método | Servicio | Descripción |
|--------|----------|-------------|
| `sdk.processScannedQR(encrypted)` | MyCashlessSDK | Desencripta, valida, aplica tx y sincroniza |
| `sdk.syncNow()` | MyCashlessSDK | Se ejecuta automáticamente después del QR |

### Flujo 7: Generar QR del Dispositivo

```typescript
import {
  MyCashlessSDK,
  CachedData,
  getMyCashlessId,
  getEventDChipId,
  AESCrypt,
  StorageKeys,
  DChipQRCodeType,
} from '@mycashless/react-native-sdk';

async function generateDeviceQR(): Promise<string> {
  const sdk = MyCashlessSDK.getInstance();
  const eventId = await CachedData.getEventId();
  const deviceUDID = await CachedData.getDeviceUDID();

  // ── IDs del dispositivo ──
  const dchipId = getEventDChipId(eventId, deviceUDID);
  const myCashlessId = getMyCashlessId(eventId, deviceUDID);

  // ── Obtener datos del wallet ──
  const balance = sdk.walletService.getBalance();
  const tokens = sdk.walletService.getTokens();

  // ── Construir payload QR ──
  const fields = [
    String(eventId),
    String(DChipQRCodeType.SALE),    // tipo de QR
    dchipId,
    String(balance.balance),
    String(balance.promo),
    JSON.stringify(tokens),
    myCashlessId,
    // ... campos adicionales según el tipo
  ];

  const payload = fields.join('|');

  // ── Encriptar con AES ──
  const eventOption = await CachedData.getString(StorageKeys.EVENT_OPTION, '');
  const encrypted = AESCrypt.encrypt(eventOption, payload);

  return encrypted; // Usar con cualquier librería QR para renderizar
}
```

**Métodos involucrados:**

| Método | Módulo |
|--------|--------|
| `CachedData.getEventId()` | CachedData |
| `CachedData.getDeviceUDID()` | CachedData |
| `getEventDChipId(eventId, udid)` | DeviceUtil |
| `getMyCashlessId(eventId, udid)` | DeviceUtil |
| `walletService.getBalance()` | WalletService |
| `walletService.getTokens()` | WalletService |
| `AESCrypt.encrypt(key, data)` | AESCrypt |
| `CachedData.getString(key)` | CachedData |

### Flujo 7b: Mostrar QR del Usuario en la UI

Para renderizar el QR en pantalla, usa la librería `qrcode` (JavaScript puro, sin dependencias nativas):

```bash
npm install qrcode
npm install --save-dev @types/qrcode
```

#### Componente QRCodeView (renderizador puro con Views)

```tsx
import React from 'react';
import { View } from 'react-native';

const QR_CELL_SIZE = 5; // Tamaño de cada celda en pixeles

const QRCodeView: React.FC<{ modules: boolean[][] }> = ({ modules }) => {
  const size = modules.length * QR_CELL_SIZE;
  return (
    <View style={{ width: size, height: size, flexDirection: 'row', flexWrap: 'wrap' }}>
      {modules.map((row, y) =>
        row.map((cell, x) => (
          <View
            key={`${y}-${x}`}
            style={{
              width: QR_CELL_SIZE,
              height: QR_CELL_SIZE,
              backgroundColor: cell ? '#000' : '#fff',
            }}
          />
        )),
      )}
    </View>
  );
};
```

#### Generar y mostrar el QR completo

```tsx
import React, { useState, useCallback, useEffect } from 'react';
import { View, Text, StyleSheet } from 'react-native';
import QRCodeLib from 'qrcode';
import {
  ChipData, CachedData, AESCrypt, StorageKeys,
  DChipQRCodeType, getEventDChipId,
} from '@mycashless/react-native-sdk';

function AccountScreen() {
  const [qrModules, setQrModules] = useState<boolean[][] | null>(null);

  const generateQR = useCallback(async () => {
    // 1. Construir el payload del QR (datos del wallet + usuario)
    const eventId = await CachedData.getEventId();
    if (!eventId) { setQrModules(null); return; }

    const chipData = ChipData.getInstance();
    const state = chipData.getState();
    const tokens = chipData.getTokens();
    const deviceUDID = await CachedData.getDeviceUDID();
    const dchipId = getEventDChipId(eventId, deviceUDID);

    await chipData.saveData();
    const nData = await CachedData.getString(StorageKeys.CHIP_DATA, '');

    const firstName = await CachedData.getString(StorageKeys.USER_FIRST_NAME, '');
    const lastName = await CachedData.getString(StorageKeys.USER_LAST_NAME, '');
    const phone = await CachedData.getString(StorageKeys.USER_PHONE, '');
    const email = await CachedData.getString(StorageKeys.USER_EMAIL, '');
    const usercode = await CachedData.getString(StorageKeys.USER_CODE, '');
    const tipPercent = await CachedData.getInt(StorageKeys.AUTO_TIP, 0);

    const raw = [
      eventId, DChipQRCodeType.PERSONAL_QR, dchipId,
      state.balance, state.promo, JSON.stringify(tokens), nData,
      firstName, lastName, phone, email, usercode,
      0, '', tipPercent,
    ].join('|');

    // 2. Encriptar con AES
    const eKey = await CachedData.getString(StorageKeys.EVENT_OPTION, '');
    const encrypted = eKey ? AESCrypt.encrypt(eKey, raw) : raw;

    // 3. Generar la matriz de módulos QR
    const qrData = QRCodeLib.create(encrypted, { errorCorrectionLevel: 'M' });
    const size = qrData.modules.size;
    const modules: boolean[][] = [];
    for (let y = 0; y < size; y++) {
      const row: boolean[] = [];
      for (let x = 0; x < size; x++) {
        row.push(qrData.modules.get(x, y) === 1);
      }
      modules.push(row);
    }
    setQrModules(modules);
  }, []);

  useEffect(() => { generateQR(); }, [generateQR]);

  return (
    <View style={styles.container}>
      {qrModules ? (
        <View style={styles.qrWrapper}>
          <QRCodeView modules={qrModules} />
        </View>
      ) : (
        <View style={styles.qrPlaceholder}>
          <Text>No QR Available</Text>
        </View>
      )}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { alignItems: 'center', padding: 16 },
  qrWrapper: {
    padding: 16,
    backgroundColor: '#fff',
    borderRadius: 12,
    // Sombra para resaltar el QR
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 4,
    elevation: 3,
  },
  qrPlaceholder: {
    width: 200,
    height: 200,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#f0f0f0',
    borderRadius: 12,
  },
});
```

**Dependencias:**

| Paquete | Propósito |
|---------|-----------|
| `qrcode` | Genera la matriz de módulos QR (JS puro, sin nativo) |
| `@types/qrcode` | Tipos TypeScript (devDependency) |

**Notas:**
- `QR_CELL_SIZE` controla el tamaño visual del QR. 5px es bueno para pantallas móviles; aumenta a 8-10 para QR más grandes.
- La librería `qrcode` es 100% JavaScript — no requiere linking nativo ni pod install.
- El QR se debe regenerar cada vez que cambie el balance, promo o tokens del usuario (llamar `generateQR()` después de `updateView()`).

### Flujo 8: Historial de Transacciones

```typescript
import {
  MyCashlessSDK,
  TransactionType,
  TransactionStatus,
  CachedData,
  StorageKeys,
} from '@mycashless/react-native-sdk';
import type { Transaction } from '@mycashless/react-native-sdk';

const sdk = MyCashlessSDK.getInstance();

async function loadTransactionHistory() {
  // ── Obtener marca de moneda del evento ──
  const currencyMark = await CachedData.getString(StorageKeys.EVENT_CURRENCY_MARK, '');

  // ── Cargar transacciones (ordenadas por created_time DESC) ──
  const all = await sdk.walletService.getTransactionHistory();

  // ── Filtrar TICKET_SALE (como las apps nativas) ──
  const transactions = all.filter(
    tx => tx.type !== TransactionType.TICKET_SALE,
  );

  // ── Mostrar cada transacción ──
  transactions.forEach(tx => {
    const typeLabel = getTypeLabel(tx.type);
    const amount = (tx.balance / 100).toFixed(2);
    const isCanceled = tx.status === TransactionStatus.CANCELED
                    || tx.status === TransactionStatus.REMOVED;

    console.log(`${typeLabel}: $${amount} ${currencyMark}`);
    if (tx.promo > 0) console.log(`  Promo: ${(tx.promo / 100).toFixed(2)}`);
    if (tx.operator_name) console.log(`  Operador: ${tx.operator_name}`);
    if (isCanceled) console.log('  ** CANCELADA **');
    if (!tx.is_synced) console.log('  (pendiente de sync)');
  });
}

function getTypeLabel(type: number): string {
  switch (type) {
    case TransactionType.BALANCE:  return 'Recarga';
    case TransactionType.SALE:     return 'Compra';
    case TransactionType.TIP:      return 'Propina';
    case TransactionType.REFUND:   return 'Reembolso';
    case TransactionType.TRANSFER: return 'Transferencia';
    case TransactionType.DEPOSIT:  return 'Depósito';
    case TransactionType.DONATION: return 'Donación';
    case TransactionType.REDEEM:   return 'Canje';
    default:                       return 'Transacción';
  }
}
```

**Campos de `Transaction`:**

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `uid` | `string` | Identificador único |
| `type` | `number` | Tipo (ver TransactionType) |
| `status` | `number` | Estado (ver TransactionStatus) |
| `balance` | `number` | Monto en centavos |
| `promo` | `number` | Promo en centavos |
| `token_detail` | `string` | JSON con detalle de tokens |
| `operator_name` | `string?` | Nombre del operador/punto de venta |
| `created_time` | `string` | Timestamp ISO |
| `is_synced` | `boolean` | Si ya se sincronizó al backend |
| `payment_type` | `number` | Tipo de pago (DCHIP, CARD, etc.) |
| `currency` | `string` | Moneda |

### Flujo 9: Restaurar Sesión al Re-abrir la App

```typescript
const sdk = MyCashlessSDK.getInstance();

async function restoreSession() {
  // ── Paso 1: Inicializar SDK ──
  await sdk.initialize(config, adapters);

  // ── Paso 2: Verificar sesión activa ──
  const authService = sdk.authService;
  const isLoggedIn = await authService.isLoggedIn();

  if (!isLoggedIn) {
    // Redirigir a login
    return;
  }

  // ── Paso 3: Restaurar token en ApiClient ──
  await authService.restoreSession();

  // ── Paso 4: Verificar si hay evento conectado ──
  const eventId = await CachedData.getEventId();
  const eventUid = await CachedData.getEventUid();

  if (eventUid) {
    // Re-establecer el evento (carga ChipData y wallet desde DB)
    await sdk.setEvent(eventUid);

    // ── Paso 5: Sincronizar estado desde el servidor ──
    await sdk.syncEventConnection();
    sdk.startSync();
  }

  // ── Listo: usuario y wallet restaurados ──
  const user = await authService.getCurrentUser();
  const balance = sdk.walletService.getBalance();
}
```

### Flujo 10: Canjear Código Promocional

```typescript
const sdk = MyCashlessSDK.getInstance();
const walletService = sdk.walletService;

async function redeemPromo(code: string) {
  const result = await walletService.redeemPromotion(code);

  if (result.success) {
    // Balance y/o promo actualizado localmente
    const newBalance = walletService.getBalance();
    console.log('Nuevo balance:', newBalance.balance / 100);
    console.log('Nuevo promo:', newBalance.promo / 100);
  } else {
    console.error('Error:', result.message);
  }
}
```

### Flujo 11: Logout y Limpieza

```typescript
const sdk = MyCashlessSDK.getInstance();

async function logout() {
  // ── Detener sincronización ──
  sdk.stopSync();

  // ── Cerrar sesión (limpia token, usuario, wallet) ──
  await sdk.authService.logout();

  // ── O para limpiar TODO (incluyendo evento): ──
  // await sdk.clearAllData();
}
```

## Ejemplos de Integración (Test App)

Los siguientes ejemplos son fragmentos reales de la app de pruebas (`MyCashlessTestApp`) que muestran cómo consumir el SDK para pintar datos en la UI.

### Pantalla de Cuenta (AccountScreen)

#### Mostrar Balance, Promo, Tokens, DChip ID

```tsx
import {
  MyCashlessSDK, WalletService, CachedData, DatabaseManager,
  StorageKeys, getMyCashlessId,
} from '@mycashless/react-native-sdk';

const [balance, setBalance] = useState('0.00');
const [promo, setPromo] = useState('0.00');
const [tokenCount, setTokenCount] = useState('0');
const [accessCount, setAccessCount] = useState('0');
const [cardId, setCardId] = useState('---');
const [userName, setUserName] = useState('');

const updateView = useCallback(async () => {
  // ── Balance y promo formateados ──
  const wallet = new WalletService();
  const formatted = wallet.getBalanceFormatted();
  setBalance(formatted.balance.toFixed(2));
  setPromo(formatted.promo.toFixed(2));

  // ── Tokens ──
  const totalTokens = wallet.getTotalTokenCount();
  setTokenCount(String(totalTokens));

  // ── Accesos (transfers tipo access) ──
  const db = DatabaseManager.getInstance();
  const eventId = await CachedData.getEventId();
  if (eventId) {
    const transfers = await db.getAccessTransfers(eventId);
    setAccessCount(String(transfers.length));
  }

  // ── DChip ID (MyCashless ID) ──
  const udid = await CachedData.getDeviceUDID();
  const evId = await CachedData.getEventId();
  const fullId = evId ? getMyCashlessId(evId, udid) : udid;
  setCardId(fullId || '---');

  // ── Nombre del usuario ──
  const firstName = await CachedData.getString(StorageKeys.USER_FIRST_NAME, '');
  const lastName = await CachedData.getString(StorageKeys.USER_LAST_NAME, '');
  setUserName(`${firstName} ${lastName}`.trim());
}, []);

// ── Render ──
<View style={styles.card}>
  <Text style={styles.label}>Balance</Text>
  <Text style={styles.value}>${balance}</Text>

  <Text style={styles.label}>Promo</Text>
  <Text style={styles.value}>${promo}</Text>

  <Text style={styles.label}>Tokens</Text>
  <Text style={styles.value}>{tokenCount}</Text>

  <Text style={styles.label}>Accesses</Text>
  <Text style={styles.value}>{accessCount}</Text>

  <Text style={styles.label}>DCHIP ID</Text>
  <Text style={styles.value}>{cardId}</Text>
</View>
```

#### Generar QR del Usuario

```tsx
import {
  ChipData, CachedData, AESCrypt, StorageKeys,
  DChipQRCodeType, getEventDChipId,
} from '@mycashless/react-native-sdk';

async function buildUserQRCode(): Promise<string | null> {
  const eventId = await CachedData.getEventId();
  if (!eventId) return null;

  const chipData = ChipData.getInstance();
  const state = chipData.getState();
  const tokens = chipData.getTokens();
  const tokenDetail = JSON.stringify(tokens);

  const deviceUDID = await CachedData.getDeviceUDID();
  const dchipId = getEventDChipId(eventId, deviceUDID);

  // Guardar chipData antes de leer nData
  await chipData.saveData();
  const nData = await CachedData.getString(StorageKeys.CHIP_DATA, '');

  // Datos del usuario
  const firstName = await CachedData.getString(StorageKeys.USER_FIRST_NAME, '');
  const lastName = await CachedData.getString(StorageKeys.USER_LAST_NAME, '');
  const phone = await CachedData.getString(StorageKeys.USER_PHONE, '');
  const email = await CachedData.getString(StorageKeys.USER_EMAIL, '');
  const usercode = await CachedData.getString(StorageKeys.USER_CODE, '');
  const tipPercent = await CachedData.getInt(StorageKeys.AUTO_TIP, 0);

  // Construir payload (mismo formato que Android QRUtil.getUserQRCode())
  const raw = [
    eventId, DChipQRCodeType.PERSONAL_QR, dchipId,
    state.balance, state.promo, tokenDetail, nData,
    firstName, lastName, phone, email, usercode,
    0, '', tipPercent,
  ].join('|');

  // Encriptar con AES
  const eKey = await CachedData.getString(StorageKeys.EVENT_OPTION, '');
  return eKey ? AESCrypt.encrypt(eKey, raw) : raw;
}

// Renderizar con cualquier librería QR (ej. qrcode + View nativo)
const encrypted = await buildUserQRCode();
// → Pasar `encrypted` a tu componente QR
```

#### Inicialización + Conexión + Login + Sync

```tsx
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import { AsyncStorageAdapter, OpSqliteDatabaseAdapter } from '../adapters';

// ── Init SDK ──
const sdk = MyCashlessSDK.getInstance();
if (!sdk.isInitialized()) {
  await sdk.initialize(
    { production: true, debug: true },
    { storage: new AsyncStorageAdapter(), database: new OpSqliteDatabaseAdapter() },
  );
}

// ── Conectar a evento ──
const event = await sdk.connectByCode('mi-evento');
if (event) {
  sdk.startSync(); // Sync periódico cada 5 min
}

// ── Login ──
const result = await sdk.authService.login(phone, password);
if (result.success) {
  // Sync conexión (obtiene dchip_id, balance del servidor, config del evento)
  await sdk.syncEventConnection();
  // Actualizar la vista con los datos nuevos
  await updateView();
}

// ── Logout ──
await sdk.authService.logout();
```

#### Recarga con Selección Dinámica de Método de Pago

```tsx
import { MyCashlessSDK, PaymentService } from '@mycashless/react-native-sdk';
import type { ReloadAmount } from '@mycashless/react-native-sdk';

const sdk = MyCashlessSDK.getInstance();
const paymentService = sdk.paymentService;

// ── 1. Obtener montos de recarga ──
const amounts = await paymentService.getReloadAmounts();
// amounts = [{ id: 1, amount: 5000, promo: 500, description: "$50 + $5 promo" }, ...]

// ── 2. Construir detalle para el monto seleccionado ──
const selected = amounts[0];
const reloadDetail = paymentService.buildReloadDetail([
  { id: selected.id, quantity: 1, is_other: selected.is_other, amount: selected.amount },
]);

// ── 3. Calcular comisiones ──
const fees = await paymentService.calculateTotalWithFees(selected.amount);
// fees = { subtotal: 5000, fee: 250, total: 5250, feePercent: 5 }

// ── 4. Detectar métodos de pago disponibles ──
const methods = await paymentService.getAvailablePaymentMethods();
// methods = { stripe: true, mercadoPago: true, paypal: false, cash: false, ... }

// ── 5. Según el método elegido: ──

// Stripe:
const intent = await paymentService.createPaymentIntent(reloadDetail);
// → initPaymentSheet → presentPaymentSheet → handlePaymentSuccess

// MercadoPago:
const pref = await paymentService.createMercadoPagoPreference(reloadDetail);
await Linking.openURL(pref.initPoint);
// → usuario paga → verifyMercadoPagoPayment(pref.externalReference)

// ── 6. Post-pago — sincronizar balance ──
const syncResult = await sdk.syncNow();
if (syncResult.transfersSynced === 0) {
  await sdk.applyReloadLocally(selected.amount);
}
```

#### Escanear QR de mPOS

```tsx
import { MyCashlessSDK, QRCodeStatus } from '@mycashless/react-native-sdk';
import type { ProcessQRResult } from '@mycashless/react-native-sdk';

// Cuando la cámara detecta un QR:
const handleQRScanned = async (data: string) => {
  const sdk = MyCashlessSDK.getInstance();
  const result: ProcessQRResult = await sdk.processScannedQR(data);

  if (result.status === QRCodeStatus.VALID) {
    // Transacción procesada exitosamente
    Alert.alert(
      'QR Processed',
      `${result.message}\n` +
      `Amount: $${((result.amount || 0) / 100).toFixed(2)}\n` +
      `New Balance: $${((result.newBalance?.balance || 0) / 100).toFixed(2)}`,
    );
    await updateView(); // Refrescar UI con nuevo balance
  } else {
    // Mostrar error con detalle de diagnóstico
    Alert.alert('Scan Failed', result.message);
  }
};
```

### Pantalla de Transacciones (TransactionsScreen)

#### Cargar y Mostrar Historial Completo

```tsx
import React, { useState, useCallback } from 'react';
import { FlatList, RefreshControl } from 'react-native';
import { useFocusEffect } from '@react-navigation/native';
import {
  MyCashlessSDK, CachedData, TransactionType, TransactionStatus, StorageKeys,
} from '@mycashless/react-native-sdk';
import type { Transaction } from '@mycashless/react-native-sdk';

const [transactions, setTransactions] = useState<Transaction[]>([]);
const [currencyMark, setCurrencyMark] = useState('');
const [refreshing, setRefreshing] = useState(false);

const loadTransactions = useCallback(async () => {
  const sdk = MyCashlessSDK.getInstance();
  if (!sdk.isEventSet()) { setTransactions([]); return; }

  // Marca de moneda del evento (ej: "MXN", "USD")
  const mark = await CachedData.getString(StorageKeys.EVENT_CURRENCY_MARK, '');
  setCurrencyMark(mark);

  // Cargar todas las transacciones desde SQLite local
  const all = await sdk.walletService.getTransactionHistory();

  // Filtrar TICKET_SALE (como las apps nativas iOS/Android)
  const filtered = all.filter(tx => tx.type !== TransactionType.TICKET_SALE);
  setTransactions(filtered);
}, []);

// Recargar al enfocar la pantalla (como viewWillAppear / onResume)
useFocusEffect(
  useCallback(() => { loadTransactions(); }, [loadTransactions])
);

// Pull-to-refresh
const onRefresh = useCallback(async () => {
  setRefreshing(true);
  await loadTransactions();
  setRefreshing(false);
}, [loadTransactions]);
```

#### Renderizar Cada Fila de Transacción

```tsx
function getTypeLabel(type: number): string {
  switch (type) {
    case TransactionType.BALANCE:  return 'Reload';
    case TransactionType.SALE:     return 'Purchase';
    case TransactionType.TIP:      return 'Tip';
    case TransactionType.REFUND:   return 'Refund';
    case TransactionType.TRANSFER: return 'Transfer';
    case TransactionType.DEPOSIT:  return 'Deposit';
    case TransactionType.DONATION: return 'Donation';
    case TransactionType.REDEEM:   return 'Redeem';
    default:                       return 'Transaction';
  }
}

function getAmountColor(type: number, status: number): string {
  if (status === TransactionStatus.CANCELED || status === TransactionStatus.REMOVED) {
    return '#e53935'; // rojo para canceladas
  }
  switch (type) {
    case TransactionType.BALANCE:
    case TransactionType.REFUND:
    case TransactionType.DEPOSIT:
      return '#2e7d32'; // verde para ingresos
    case TransactionType.SALE:
    case TransactionType.TIP:
    case TransactionType.DONATION:
      return '#c62828'; // rojo para egresos
    case TransactionType.TRANSFER:
      return status === TransactionStatus.ADDED ? '#2e7d32' : '#c62828';
    default:
      return '#333';
  }
}

function formatCurrency(cents: number, currencyMark: string): string {
  const value = (cents / 100).toFixed(2);
  return `$${value} ${currencyMark}`;
}

function getTokenCount(tokenDetail: string): number {
  try {
    const parsed = JSON.parse(tokenDetail);
    return Object.values(parsed)
      .filter((v): v is number => typeof v === 'number' && v > 0)
      .reduce((sum, v) => sum + v, 0);
  } catch { return 0; }
}

const TransactionRow: React.FC<{ item: Transaction; currencyMark: string }> = ({
  item, currencyMark,
}) => {
  const isCanceled = item.status === TransactionStatus.CANCELED
                  || item.status === TransactionStatus.REMOVED;
  const amountColor = getAmountColor(item.type, item.status);
  const tokenCount = getTokenCount(item.token_detail);

  return (
    <View style={[styles.row, isCanceled && styles.rowCanceled]}>
      <View style={styles.rowLeft}>
        <Text style={[styles.typeLabel, { color: isCanceled ? '#e53935' : '#333' }]}>
          {getTypeLabel(item.type)}
        </Text>
        {item.operator_name ? (
          <Text style={styles.operator}>{item.operator_name}</Text>
        ) : null}
        <Text style={styles.timestamp}>
          {new Date(item.created_time).toLocaleString()}
        </Text>
        {!item.is_synced && (
          <Text style={styles.syncPending}>Pending sync</Text>
        )}
      </View>

      <View style={styles.rowRight}>
        <Text style={[styles.amount, { color: amountColor }]}>
          {formatCurrency(item.balance, currencyMark)}
        </Text>
        {item.promo > 0 && (
          <Text style={styles.promo}>
            Promo: {(item.promo / 100).toFixed(2)}
          </Text>
        )}
        {tokenCount > 0 && (
          <Text style={styles.tokens}>Tokens: {tokenCount}</Text>
        )}
      </View>
    </View>
  );
};
```

#### FlatList con Empty State

```tsx
<FlatList
  data={transactions}
  renderItem={({ item }) => (
    <TransactionRow item={item} currencyMark={currencyMark} />
  )}
  keyExtractor={(item) => item.uid}
  refreshControl={
    <RefreshControl refreshing={refreshing} onRefresh={onRefresh} />
  }
  ListEmptyComponent={
    <View style={styles.emptyView}>
      <Text style={styles.emptyText}>No transactions yet</Text>
      <Text style={styles.emptySubtext}>
        Transactions will appear here after QR scans or reloads
      </Text>
    </View>
  }
  ListHeaderComponent={
    transactions.length > 0 ? (
      <Text style={styles.countLabel}>
        {transactions.length} transaction{transactions.length !== 1 ? 's' : ''}
      </Text>
    ) : null
  }
/>
```

## Adaptadores de Almacenamiento

El SDK requiere adaptadores para almacenamiento seguro y base de datos. Esto permite compatibilidad con React Native CLI y Expo.

### React Native (react-native-keychain + react-native-sqlite-storage)

```typescript
import * as Keychain from 'react-native-keychain';
import SQLite from 'react-native-sqlite-storage';
import type { StorageAdapter, DatabaseAdapter, DatabaseResult } from '@mycashless/react-native-sdk';

class KeychainAdapter implements StorageAdapter {
  private readonly SERVICE = 'com.mycashless.sdk';

  async getItem(key: string): Promise<string | null> {
    const result = await Keychain.getGenericPassword({
      service: `${this.SERVICE}.${key}`,
    });
    return result ? result.password : null;
  }

  async setItem(key: string, value: string): Promise<void> {
    await Keychain.setGenericPassword(key, value, {
      service: `${this.SERVICE}.${key}`,
    });
  }

  async removeItem(key: string): Promise<void> {
    await Keychain.resetGenericPassword({ service: `${this.SERVICE}.${key}` });
  }

  async clear(): Promise<void> {
    // Implementar según necesidades
  }
}

class SQLiteAdapter implements DatabaseAdapter {
  private db!: SQLite.SQLiteDatabase;

  async open(): Promise<void> {
    this.db = await SQLite.openDatabase({ name: 'mycashless.db' });
  }

  async executeSql(sql: string, params?: unknown[]): Promise<DatabaseResult> {
    const [result] = await this.db.executeSql(sql, params as any[]);
    return {
      rows: {
        length: result.rows.length,
        item: (i) => result.rows.item(i),
        raw: () => result.rows.raw(),
      },
      insertId: result.insertId,
      rowsAffected: result.rowsAffected,
    };
  }

  async close(): Promise<void> {
    await this.db.close();
  }

  async transaction(fn: (tx: DatabaseTransaction) => Promise<void>): Promise<void> {
    await this.db.transaction(async (tx) => {
      await fn({
        executeSql: async (sql, params) => {
          const [result] = await tx.executeSql(sql, params as any[]);
          return {
            rows: { length: result.rows.length, item: (i) => result.rows.item(i), raw: () => result.rows.raw() },
            insertId: result.insertId,
            rowsAffected: result.rowsAffected,
          };
        },
      });
    });
  }
}
```

### Expo (expo-secure-store + expo-sqlite)

```typescript
import * as SecureStore from 'expo-secure-store';
import * as SQLite from 'expo-sqlite';
import type { StorageAdapter, DatabaseAdapter } from '@mycashless/react-native-sdk';

class ExpoStorageAdapter implements StorageAdapter {
  async getItem(key: string): Promise<string | null> {
    return await SecureStore.getItemAsync(key);
  }

  async setItem(key: string, value: string): Promise<void> {
    await SecureStore.setItemAsync(key, value);
  }

  async removeItem(key: string): Promise<void> {
    await SecureStore.deleteItemAsync(key);
  }

  async clear(): Promise<void> {
    // Expo SecureStore no tiene bulk clear
  }
}

// Database adapter para expo-sqlite — implementación similar a SQLiteAdapter
```

### Inicialización con Adaptadores

```typescript
await MyCashlessSDK.getInstance().initialize(
  {
    production: true,
    language: 'es',
    debug: false,
  },
  {
    storage: new KeychainAdapter(),   // o ExpoStorageAdapter
    database: new SQLiteAdapter(),    // o ExpoSQLiteAdapter
  }
);
```

## API Reference

### MyCashlessSDK

Clase principal del SDK. Singleton.

| Método | Descripción |
|--------|-------------|
| `getInstance()` | Obtiene la instancia singleton |
| `initialize(config, adapters?)` | Inicializa el SDK con configuración y adaptadores |
| `setEvent(eventUid)` | Configura el evento por UID |
| `connectByCode(nickname)` | Conecta a un evento por código/nickname |
| `syncEventConnection()` | Sincroniza la conexión del evento con el servidor |
| `getCurrentEvent()` | Obtiene info del evento actual (id, uid, nombre, moneda, etc.) |
| `startSync()` | Inicia sincronización periódica (cada 5 min) |
| `stopSync()` | Detiene sincronización periódica |
| `syncNow()` | Ejecuta sincronización inmediata — retorna `SyncResult` |
| `syncBalanceFromServer()` | Intenta sincronizar balance desde el servidor |
| `applyReloadLocally(balanceCents, promoCents?)` | Aplica una recarga localmente al wallet |
| `processScannedQR(encryptedQR)` | Procesa un QR escaneado — retorna `ProcessQRResult` |
| `isInitialized()` | Verifica si el SDK está inicializado |
| `isEventSet()` | Verifica si hay un evento configurado |
| `getConfig()` | Obtiene la configuración actual |
| `getState()` | Obtiene el estado actual (balance, promo, tokens, etc.) |
| `clearAllData()` | Limpia todos los datos locales |
| `reset()` | Resetea la instancia singleton |

**Acceso a servicios:**

```typescript
const sdk = MyCashlessSDK.getInstance();
sdk.authService;     // AuthService
sdk.walletService;   // WalletService
sdk.paymentService;  // PaymentService
sdk.syncService;     // SyncService
```

### SDKConfig

```typescript
interface SDKConfig {
  production: boolean;          // true = producción, false = staging
  stripePublishableKey?: string;
  language?: string;            // 'es', 'en', 'fr', 'pt'
  debug?: boolean;
}
```

### SDKState

```typescript
interface SDKState {
  initialized: boolean;   // SDK fue inicializado
  eventSet: boolean;      // Evento configurado con setEvent()
  production: boolean;    // true = producción, false = staging
}
```

## React Hooks

### useAuth

```typescript
const {
  // Estado
  user,            // Partial<User> | null
  isLoggedIn,      // boolean
  isLoading,       // boolean
  error,           // string | null

  // Acciones
  login,           // (phone, password) => Promise<AuthResult>
  register,        // (userData, firebaseToken?) => Promise<AuthResult>
  logout,          // () => Promise<void>
  updateUser,      // (userData) => Promise<User | null>
  refreshUser,     // () => Promise<void>
  checkPhoneExists, // (phone) => Promise<boolean>
} = useAuth();
```

### useWallet

```typescript
const {
  // Balance (en centavos)
  balance,          // number
  promo,            // number
  total,            // number (balance + promo)
  tokens,           // TokenAmount
  reload,           // number
  firstReloaded,    // boolean

  // Formateado
  balanceFormatted, // string ("$100.00")
  promoFormatted,   // string
  totalFormatted,   // string

  // Estado
  isLoading,        // boolean
  error,            // string | null

  // Acciones
  refreshWallet,         // () => Promise<void>
  getTransactionHistory, // () => Promise<Transaction[]>
  hasSufficientBalance,  // (amount, usePromo?) => boolean
} = useWallet('MXN');
```

### usePayment

```typescript
const {
  // Estado
  isLoading,        // boolean
  error,            // string | null
  paymentMethods,   // SavedPaymentMethod[]
  reloadAmounts,    // ReloadAmount[]
  availableMethods, // AvailablePaymentMethods | null

  // Configuración
  loadConfig,              // () => Promise<void>
  getStripePublishableKey, // () => Promise<string>
  getApplePayMerchantId,   // () => string

  // Stripe
  createPaymentIntent,   // (reloadDetail, customerId?, paymentMethodId?, referralCode?) => Promise<PaymentIntentResult | null>
  confirmPaymentIntent,  // (paymentIntentId) => Promise<boolean>
  applyReloadToChip,     // () => Promise<boolean>

  // Métodos de pago guardados
  loadPaymentMethods,    // () => Promise<void>
  createSetupIntent,     // () => Promise<SetupIntentResult | null>
  deletePaymentMethod,   // (methodId) => Promise<boolean>

  // Billing
  getBillingInfo,        // () => Promise<BillingInfo | null>
  saveBillingInfo,       // (info) => Promise<boolean>

  // PayPal
  getPayPalClientId,     // () => Promise<string | null>
  createPayPalOrder,     // (reloadDetail, referralCode?) => Promise<PayPalOrderResult | null>
  capturePayPalOrder,    // (orderId, transferIds, tableIds, serviceFee) => Promise<...>

  // Utilidades
  calculateTotal,        // (amount) => Promise<{ subtotal, fee, total, feePercent }>
  buildReloadDetail,     // (amounts) => string
  formatAmount,          // (amountInCents) => string
} = usePayment('MXN');
```

### useSync

```typescript
const {
  // Estado
  isSyncing,       // boolean
  lastSyncTime,    // Date | null
  lastSyncResult,  // SyncResult | null
  error,           // string | null

  // Acciones
  syncNow,         // () => Promise<SyncResult>
  startAutoSync,   // () => void
  stopAutoSync,    // () => void
} = useSync();
```

## Servicios

### AuthService

| Método | Descripción |
|--------|-------------|
| `login(phone, password)` | Inicia sesión con teléfono y contraseña |
| `loginWithFankiToken(fankiToken)` | Login/registro con token de Fanki (sin password) |
| `getFankiEvents()` | Obtiene eventos activos asociados a la organización Fanki |
| `register(userData, firebaseToken?)` | Registra un nuevo usuario |
| `resetPassword(phone, newPassword, firebaseToken?)` | Restablece la contraseña |
| `logout()` | Cierra sesión y limpia datos |
| `checkPhoneExists(phone)` | Verifica si un teléfono está registrado |
| `startWhatsappVerification(phone)` | Inicia verificación por WhatsApp |
| `checkWhatsappVerification(phone, code)` | Verifica código de WhatsApp |
| `updateUser(userData)` | Actualiza datos del usuario |
| `deleteAccount()` | Elimina la cuenta del usuario |
| `isLoggedIn()` | Verifica si hay sesión activa |
| `getCurrentUser()` | Obtiene datos del usuario actual |
| `isSessionExpired()` | Verifica si la sesión expiró |
| `restoreSession()` | Restaura sesión desde almacenamiento |

### WalletService

| Método | Descripción |
|--------|-------------|
| `getBalance()` | Obtiene balance actual (balance, promo, tokens) |
| `getBalanceFormatted()` | Balance con formato (balance, promo, total, tokens) |
| `getReloadAmounts(eventUid)` | Montos de recarga disponibles |
| `getTransactionHistory()` | Historial completo de transacciones |
| `getRecentTransactions(limit?)` | Últimas N transacciones (default: 10) |
| `createTransaction(params)` | Crea una transacción local |
| `recordReload(amount, promo?, tokens?)` | Registra una recarga |
| `redeemPromotion(code)` | Canjea un código promocional |
| `hasSufficientBalance(amount, usePromo?)` | Verifica si hay balance suficiente |
| `calculatePaymentBreakdown(amount)` | Calcula desglose balance/promo para un monto |
| `loadWallet()` | Carga wallet desde almacenamiento |
| `saveWallet()` | Guarda wallet a almacenamiento |
| `clearWallet()` | Limpia el wallet |
| `getTokens()` | Obtiene tokens disponibles |
| `getTotalTokenCount()` | Cuenta total de tokens |

### PaymentService

| Método | Descripción |
|--------|-------------|
| `getAvailablePaymentMethods()` | Métodos de pago disponibles para el evento |
| `getReloadAmounts()` | Montos de recarga configurados |
| `getPayableConfig()` | Configuración de cobros del evento |
| `getStripePublishableKey()` | Llave pública de Stripe |
| `getConnectedAccountId()` | ID de cuenta conectada de Stripe |
| `getApplePayMerchantId()` | Merchant ID para Apple Pay |
| `createPaymentIntent(reloadDetail, ...)` | Crea un PaymentIntent en Stripe |
| `confirmPaymentIntent(paymentIntentId)` | Confirma un PaymentIntent |
| `applyReloadToChip()` | Aplica recarga pendiente al chip |
| `createSetupIntent()` | Crea SetupIntent para guardar método de pago |
| `getPaymentMethods()` | Lista métodos de pago guardados |
| `deletePaymentMethod(id)` | Elimina un método de pago guardado |
| `getBillingInfo()` | Obtiene info de facturación |
| `saveBillingInfo(info)` | Guarda info de facturación |
| `getPayPalClientId()` | Client ID de PayPal |
| `createPayPalOrder(reloadDetail, ...)` | Crea orden de PayPal |
| `capturePayPalOrder(orderId, ...)` | Captura orden de PayPal |
| `createMercadoPagoPreference(reloadDetail, referralCode?)` | Crea preferencia de MercadoPago — retorna URL de checkout |
| `verifyMercadoPagoPayment(externalRef?, maxRetries?)` | Verifica pago de MercadoPago (con reintentos exponenciales) |
| `getMercadoPagoErrorMessage(statusDetail)` | Mensaje de error legible para un código de MercadoPago |
| `shouldSuggestDifferentMethod(statusDetail)` | Si el error sugiere usar otro método de pago |
| `calculateTotalWithFees(amount)` | Calcula total con comisiones |
| `buildReloadDetail(amounts)` | Construye string de detalle de recarga |
| `formatAmount(cents, currency?)` | Formatea centavos a moneda |

### SyncService

| Método | Descripción |
|--------|-------------|
| `getInstance()` | Obtiene instancia singleton |
| `setCallbacks(onComplete?, onError?)` | Configura callbacks de sync |
| `startPeriodicSync()` | Inicia sync periódico (cada 5 min) |
| `stopPeriodicSync()` | Detiene sync periódico |
| `isSyncRunning()` | Verifica si hay sync en progreso |
| `syncNow()` | Ejecuta sync inmediato — retorna `SyncResult` |
| `getLastSyncTime()` | Obtiene timestamp del último sync |

El sync sube transacciones, transfers y check-ins al backend. Después de un QR exitoso, `syncNow()` se ejecuta automáticamente en background.

#### Callback de Sync en Background

Cuando el SDK procesa un QR, ejecuta sync automáticamente en background (fire-and-forget). Para recibir notificación del resultado:

```typescript
import { MyCashlessSDK } from '@mycashless/react-native-sdk';
import type { SyncStatusCallback } from '@mycashless/react-native-sdk';

const sdk = MyCashlessSDK.getInstance();

sdk.setSyncStatusCallback((result) => {
  if (result.success) {
    console.log(`Sync completado: ${result.transactionsSynced} transacciones sincronizadas`);
  } else {
    console.warn('Sync falló:', result.errors);
  }
});

// Para remover el callback:
sdk.setSyncStatusCallback(null);
```

## Tipos de Transacción

| Tipo | Valor | Descripción |
|------|-------|-------------|
| `BALANCE` | 1 | Recarga de balance |
| `SALE` | 2 | Compra/venta |
| `TIP` | 3 | Propina |
| `REFUND` | 4 | Reembolso |
| `TRANSFER` | 5 | Transferencia entre usuarios |
| `DEPOSIT` | 6 | Depósito |
| `DONATION` | 9 | Donación |
| `REDEEM` | 11 | Canje de promoción |
| `TICKET_SALE` | 12 | Venta de ticket/acceso |

## Constantes

El SDK exporta las siguientes constantes:

```typescript
import {
  // URLs
  API_URLS,              // URLs de producción y staging
  LEGAL_URLS,            // URLs legales

  // Enums de transacciones
  TransactionType,       // BALANCE, SALE, TIP, REFUND, etc.
  TransactionStatus,     // CREATED, SYNCED, etc.
  PaymentType,           // DCHIP, CARD, PAYPAL, etc.
  TransferType,          // Tipos de transferencia
  TransferStatus,        // Estados de transferencia

  // Eventos
  EventType,             // Tipos de evento
  EventTypeInt,          // Tipos de evento (numérico)
  EventReloadStatus,     // Estado de recarga del evento

  // QR
  QRCodeType,            // Tipos de QR
  QRCodeStatus,          // VALID, INVALID, EXPIRED, etc.
  DChipQRCodeType,       // Tipos de QR del DChip

  // Verificación
  VerificationType,
  VerificationMethod,
  VerifyAction,

  // Pagos
  PurchaseType,
  CheckoutPaymentType,
  PaymentStatus,
  AutoTipType,
  MainReloadType,

  // Stripe
  STRIPE_KEYS,
  STRIPE_AUTO_REFUND_WORKFLOW,

  // Sync
  SyncServiceCommand,
  SyncCommand,
  SyncTransactionStatus,
  CacheServiceCommand,

  // Check-in
  CheckInType,

  // Config
  DEFAULTS,
  SYNC_CONFIG,           // Intervalos y timeouts de sync
  CHIP_LIMITS,           // Límites del chip
  AES_CONFIG,            // Configuración AES
  RETRY_CONFIG,          // Configuración de reintentos
  TIMING,                // Constantes de tiempo
  DCHIP_CONFIG,          // Configuración del DChip
  RATING_CONFIG,         // Configuración de rating
  DB_CONFIG,             // Configuración de base de datos
  StorageKeys,           // Llaves de almacenamiento
  TableNames,            // Nombres de tablas

  // Misc
  Gender,
  AppIcon,
  IconEventUID,
  NotificationAction,
  TEST_EVENT_IDS,
} from '@mycashless/react-native-sdk';
```

## Utilidades

```typescript
import {
  // Device
  generateDeviceUDID,
  generateDeviceUniqueId,
  generateTransactionUID,
  generateTransferUID,
  generateCheckInUID,
  getMyCashlessId,
  getEventDChipId,
  byteArrayToHexString,
  hexStringToByteArray,
  getPlatform,
  getDeviceInfo,

  // Tiempo
  getCurrentTimeISO,
  getCurrentTimeISONoMs,
  parseISO,
  formatToISO,
  getUnixTimestamp,
  getUnixTimestampMs,
  formatForDisplay,
  formatShortDate,
  formatTime,
  isExpired,
  getTimeDifference,
  addHours,
  addDays,
  isSameDay,
  startOfDay,
  endOfDay,
} from '@mycashless/react-native-sdk';
```

## Logger

El SDK incluye un sistema de logging estructurado. Los logs de nivel `debug`, `info` y `warn` solo se emiten cuando `debug: true` en la configuración del SDK. Los logs de nivel `error` siempre se emiten.

```typescript
import { Logger, LogLevel } from '@mycashless/react-native-sdk';
import type { LogHandler } from '@mycashless/react-native-sdk';
```

### Configuración automática

El Logger se configura automáticamente al inicializar el SDK:

```typescript
MyCashlessSDK.initialize({
  production: false,
  debug: true,  // Habilita logs de debug/info/warn
});
```

### Custom Log Handler

Para integrar con servicios de monitoreo (Sentry, Datadog, etc.):

```typescript
Logger.setLogHandler((level, tag, message, data) => {
  if (level >= LogLevel.ERROR) {
    Sentry.captureMessage(`[${tag}] ${message}`, {
      level: 'error',
      extra: { data },
    });
  }
});

// Para restaurar el logging por consola:
Logger.setLogHandler(null);
```

### Niveles de Log

| Nivel | Valor | Requiere `debug: true` |
|-------|-------|----------------------|
| `LogLevel.DEBUG` | 0 | Sí |
| `LogLevel.INFO` | 1 | Sí |
| `LogLevel.WARN` | 2 | Sí |
| `LogLevel.ERROR` | 3 | No (siempre se emite) |
| `LogLevel.NONE` | 4 | — |

## Testing

El SDK incluye 186 unit tests que cubren criptografía, API, y los tres servicios principales.

### Ejecutar tests

```bash
npm test

# Watch mode
npm run test:watch

# Con coverage
npm test -- --coverage
```

### Cobertura de tests

| Suite | Archivo | Tests |
|-------|---------|-------|
| AESCrypt | `__tests__/crypto/AESCrypt.test.ts` | 20 |
| ChipData | `__tests__/crypto/ChipData.test.ts` | 48 |
| ApiClient | `__tests__/api/ApiClient.test.ts` | 21 |
| AuthService | `__tests__/services/AuthService.test.ts` | 27 |
| WalletService | `__tests__/services/WalletService.test.ts` | 22 |
| PaymentService | `__tests__/services/PaymentService.test.ts` | 48 |

## Seguridad

- **Wallet encriptado**: ChipData almacena el estado del wallet en 24 bytes encriptados con AES-256-CBC
- **Almacenamiento seguro**: Tokens y datos sensibles se guardan via Keychain (iOS) / SecureStore (Android/Expo)
- **HTTPS**: Toda comunicación con el backend usa HTTPS
- **Retry automático en 401**: Cuando el backend responde con 401, el SDK restaura el token desde almacenamiento persistente y reintenta la petición una vez. Solo si el reintento también falla con 401 se marca la sesión como expirada y se notifica vía `onSessionExpired`
- **QR firmado**: Los códigos QR de transacción se encriptan/desencriptan con AES

## Soporte

- Issues: https://github.com/Cashless-Media/sdk-react-native/issues
- Email: soporte@mycashless.com

## Licencia

MIT
