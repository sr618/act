# **Phase 8: Mobile Application (Flutter)**
**Duration:** 12 Weeks  
**Team:** Mobile (2-3), Backend (1), QA (1)  
**Dependencies:** Phase 0-7 (All prior modules with stable APIs)  

---

## **1. Overview**

The Mobile Application provides:
- Full offline-first architecture
- Core accounting features on the go
- Real-time sync with conflict resolution
- Biometric authentication
- Invoice capture via camera (OCR)

---

## **2. Target Platforms**

| Platform | Minimum Version |
|----------|-----------------|
| Android | 8.0 (API 26) |
| iOS | 14.0 |

---

## **3. Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│                     Flutter App Layer                        │
├──────────────────────────────────────────────────────────────┤
│  Presentation Layer (Riverpod + Widgets)                     │
│  ├── Pages (Screens)                                         │
│  ├── Widgets (Reusable Components)                           │
│  └── State Management (Riverpod Providers)                   │
├──────────────────────────────────────────────────────────────┤
│  Domain Layer (Pure Dart)                                    │
│  ├── Entities                                                │
│  ├── Use Cases                                               │
│  └── Repository Interfaces                                   │
├──────────────────────────────────────────────────────────────┤
│  Data Layer                                                  │
│  ├── Local Database (Drift/SQLite)                           │
│  ├── Remote API (Dio + Retrofit)                             │
│  └── Sync Engine                                             │
└──────────────────────────────────────────────────────────────┘
```

---

## **4. Core Features (MVP)**

### **4.1 Authentication**
- [ ] Email/Password login
- [ ] Biometric authentication (Face ID, Fingerprint)
- [ ] Multi-company selection
- [ ] Secure token storage (Flutter Secure Storage)

### **4.2 Dashboard**
- [ ] KPI cards (Sales, Receivables, Cash)
- [ ] Sales trend chart
- [ ] Today's transactions summary
- [ ] Quick action buttons

### **4.3 Voucher Entry**
- [ ] Receipt voucher
- [ ] Payment voucher
- [ ] Sales invoice (simplified)
- [ ] Purchase entry

### **4.4 Party Management**
- [ ] View party list
- [ ] Party ledger statement
- [ ] Quick call/WhatsApp to party
- [ ] Outstanding amount display

### **4.5 Inventory (View Only)**
- [ ] Stock summary
- [ ] Item-wise stock
- [ ] Godown-wise stock
- [ ] Low stock alerts

### **4.6 Reports (View Only)**
- [ ] Day book
- [ ] Cash/Bank book
- [ ] Outstanding receivables
- [ ] Outstanding payables

---

## **5. Offline-First Architecture**

```dart
// lib/core/sync/sync_engine.dart
class SyncEngine {
  final LocalDatabase _localDb;
  final RemoteApiClient _apiClient;
  final ConnectivityService _connectivity;
  
  // Sync queue for pending changes
  late final Box<SyncQueueItem> _syncQueue;
  
  Future<void> initialize() async {
    _syncQueue = await Hive.openBox<SyncQueueItem>('sync_queue');
    
    // Listen to connectivity changes
    _connectivity.onConnectivityChanged.listen((status) {
      if (status == ConnectivityStatus.online) {
        _processSyncQueue();
      }
    });
  }
  
  // Called when any local change is made
  Future<void> queueForSync(SyncQueueItem item) async {
    await _syncQueue.add(item);
    
    if (await _connectivity.isOnline) {
      await _processSyncQueue();
    }
  }
  
  Future<void> _processSyncQueue() async {
    for (final item in _syncQueue.values.toList()) {
      try {
        await _syncItem(item);
        await item.delete();
      } catch (e) {
        if (e is ConflictException) {
          await _handleConflict(item, e);
        } else {
          // Keep in queue for retry
          item.retryCount++;
          await item.save();
        }
      }
    }
  }
  
  Future<void> _handleConflict(SyncQueueItem item, ConflictException e) async {
    // Conflict resolution strategy
    // 1. For vouchers: Server wins (auditable)
    // 2. For drafts: Last writer wins
    // 3. For party data: Merge if possible
    
    switch (item.conflictStrategy) {
      case ConflictStrategy.serverWins:
        await _localDb.applyServerData(item.entityType, e.serverVersion);
        await item.delete();
        break;
      case ConflictStrategy.clientWins:
        await _apiClient.forceUpdate(item.entityType, item.entityId, item.data);
        await item.delete();
        break;
      case ConflictStrategy.manual:
        await _notifyConflict(item, e);
        break;
    }
  }
  
  // Full sync from server
  Future<void> fullSync() async {
    final lastSyncTime = await _localDb.getLastSyncTime();
    
    // Get changes since last sync
    final changes = await _apiClient.getChanges(since: lastSyncTime);
    
    await _localDb.applyChanges(changes);
    await _localDb.setLastSyncTime(DateTime.now());
  }
}

class SyncQueueItem extends HiveObject {
  final String entityType;
  final String entityId;
  final String operation;  // create, update, delete
  final Map<String, dynamic> data;
  final DateTime timestamp;
  final ConflictStrategy conflictStrategy;
  int retryCount = 0;
}

enum ConflictStrategy { serverWins, clientWins, manual }
```

---

## **6. Local Database (Drift)**

```dart
// lib/data/local/database.dart
import 'package:drift/drift.dart';
import 'package:drift/native.dart';

part 'database.g.dart';

// Tables
class Vouchers extends Table {
  TextColumn get id => text()();
  TextColumn get tenantId => text()();
  TextColumn get voucherNumber => text()();
  DateTimeColumn get voucherDate => dateTime()();
  TextColumn get voucherType => text()();
  TextColumn get narration => text().nullable()();
  RealColumn get totalDebit => real()();
  RealColumn get totalCredit => real()();
  TextColumn get status => text()();
  BoolColumn get isSynced => boolean().withDefault(const Constant(false))();
  DateTimeColumn get localModifiedAt => dateTime()();
  
  @override
  Set<Column> get primaryKey => {id};
}

class VoucherLines extends Table {
  TextColumn get id => text()();
  TextColumn get voucherId => text().references(Vouchers, #id)();
  TextColumn get ledgerId => text()();
  TextColumn get ledgerName => text()();  // Denormalized for offline
  RealColumn get debit => real()();
  RealColumn get credit => real()();
  IntColumn get sequenceNumber => integer()();
  
  @override
  Set<Column> get primaryKey => {id};
}

class Ledgers extends Table {
  TextColumn get id => text()();
  TextColumn get tenantId => text()();
  TextColumn get name => text()();
  TextColumn get accountGroupId => text()();
  TextColumn get accountGroupName => text()();  // Denormalized
  RealColumn get openingBalance => real()();
  RealColumn get closingBalance => real()();
  TextColumn get gstinNumber => text().nullable()();
  TextColumn get phone => text().nullable()();
  TextColumn get email => text().nullable()();
  BoolColumn get isSynced => boolean().withDefault(const Constant(true))();
  
  @override
  Set<Column> get primaryKey => {id};
}

class Items extends Table {
  TextColumn get id => text()();
  TextColumn get tenantId => text()();
  TextColumn get code => text()();
  TextColumn get name => text()();
  TextColumn get categoryName => text().nullable()();
  TextColumn get unit => text()();
  RealColumn get mrp => real()();
  RealColumn get sellingPrice => real()();
  RealColumn get stockQuantity => real()();  // Local cache
  BoolColumn get isSynced => boolean().withDefault(const Constant(true))();
  
  @override
  Set<Column> get primaryKey => {id};
}

@DriftDatabase(tables: [Vouchers, VoucherLines, Ledgers, Items])
class AppDatabase extends _$AppDatabase {
  AppDatabase() : super(_openConnection());
  
  @override
  int get schemaVersion => 1;
  
  // Voucher operations
  Future<List<VoucherWithLines>> getVouchers({
    required String tenantId,
    DateTime? fromDate,
    DateTime? toDate,
    String? voucherType,
  }) async {
    final query = select(vouchers)
      ..where((v) => v.tenantId.equals(tenantId));
    
    if (fromDate != null) {
      query.where((v) => v.voucherDate.isBiggerOrEqualValue(fromDate));
    }
    if (toDate != null) {
      query.where((v) => v.voucherDate.isSmallerOrEqualValue(toDate));
    }
    if (voucherType != null) {
      query.where((v) => v.voucherType.equals(voucherType));
    }
    
    final voucherList = await query.get();
    
    // Get lines for each voucher
    return Future.wait(voucherList.map((v) async {
      final lines = await (select(voucherLines)
        ..where((l) => l.voucherId.equals(v.id))
        ..orderBy([(l) => OrderingTerm.asc(l.sequenceNumber)]))
        .get();
      
      return VoucherWithLines(voucher: v, lines: lines);
    }));
  }
  
  Future<void> saveVoucher(VoucherWithLines voucherWithLines) async {
    await batch((batch) {
      batch.insert(vouchers, voucherWithLines.voucher, 
          mode: InsertMode.insertOrReplace);
      
      for (final line in voucherWithLines.lines) {
        batch.insert(voucherLines, line, 
            mode: InsertMode.insertOrReplace);
      }
    });
  }
  
  // Ledger operations
  Future<List<Ledger>> searchLedgers(String query, {int limit = 20}) {
    return (select(ledgers)
      ..where((l) => l.name.like('%$query%'))
      ..limit(limit))
      .get();
  }
  
  Future<LedgerStatement> getLedgerStatement(
    String ledgerId, 
    DateTime fromDate, 
    DateTime toDate,
  ) async {
    final lines = await (select(voucherLines)
      ..where((l) => l.ledgerId.equals(ledgerId)))
      .join([
        innerJoin(vouchers, vouchers.id.equalsExp(voucherLines.voucherId))
      ])
      .get();
    
    // Calculate running balance
    // ...
  }
}

LazyDatabase _openConnection() {
  return LazyDatabase(() async {
    final dbFolder = await getApplicationDocumentsDirectory();
    final file = File(p.join(dbFolder.path, 'auditflow.sqlite'));
    return NativeDatabase(file);
  });
}
```

---

## **7. State Management (Riverpod)**

```dart
// lib/presentation/providers/voucher_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'voucher_provider.g.dart';

@riverpod
class VoucherNotifier extends _$VoucherNotifier {
  @override
  FutureOr<VoucherState> build() async {
    return VoucherState.initial();
  }
  
  Future<void> loadVouchers({
    DateTime? fromDate,
    DateTime? toDate,
    String? voucherType,
  }) async {
    state = const AsyncValue.loading();
    
    try {
      final vouchers = await ref.read(voucherRepositoryProvider).getVouchers(
        fromDate: fromDate,
        toDate: toDate,
        voucherType: voucherType,
      );
      
      state = AsyncValue.data(VoucherState(
        vouchers: vouchers,
        isLoading: false,
      ));
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }
  
  Future<void> saveVoucher(VoucherEntry entry) async {
    final repository = ref.read(voucherRepositoryProvider);
    final syncEngine = ref.read(syncEngineProvider);
    
    // Save locally first
    final voucher = await repository.saveLocal(entry);
    
    // Queue for sync
    await syncEngine.queueForSync(SyncQueueItem(
      entityType: 'voucher',
      entityId: voucher.id,
      operation: 'create',
      data: voucher.toJson(),
      conflictStrategy: ConflictStrategy.manual,
    ));
    
    // Refresh list
    await loadVouchers();
  }
}

@riverpod
class DashboardNotifier extends _$DashboardNotifier {
  @override
  FutureOr<DashboardData> build() async {
    return _fetchDashboard();
  }
  
  Future<DashboardData> _fetchDashboard() async {
    final repository = ref.read(dashboardRepositoryProvider);
    
    // Try local first, then sync from server
    final localData = await repository.getLocalDashboard();
    
    if (await ref.read(connectivityProvider).isOnline) {
      try {
        final serverData = await repository.fetchFromServer();
        return serverData;
      } catch (e) {
        // Fall back to local
        return localData;
      }
    }
    
    return localData;
  }
  
  Future<void> refresh() async {
    state = const AsyncValue.loading();
    state = AsyncValue.data(await _fetchDashboard());
  }
}

// lib/presentation/providers/auth_provider.dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  FutureOr<AuthState> build() async {
    return _checkAuthState();
  }
  
  Future<AuthState> _checkAuthState() async {
    final storage = ref.read(secureStorageProvider);
    final token = await storage.read(key: 'auth_token');
    
    if (token == null) {
      return AuthState.unauthenticated();
    }
    
    // Validate token
    try {
      final user = await ref.read(authRepositoryProvider).validateToken(token);
      return AuthState.authenticated(user: user, token: token);
    } catch (e) {
      await storage.delete(key: 'auth_token');
      return AuthState.unauthenticated();
    }
  }
  
  Future<void> login(String email, String password) async {
    state = const AsyncValue.loading();
    
    try {
      final result = await ref.read(authRepositoryProvider).login(email, password);
      
      // Store token securely
      await ref.read(secureStorageProvider).write(
        key: 'auth_token',
        value: result.token,
      );
      
      state = AsyncValue.data(AuthState.authenticated(
        user: result.user,
        token: result.token,
      ));
      
      // Trigger initial sync
      await ref.read(syncEngineProvider).fullSync();
    } catch (e, st) {
      state = AsyncValue.error(e, st);
    }
  }
  
  Future<bool> authenticateWithBiometrics() async {
    final localAuth = LocalAuthentication();
    
    final canAuth = await localAuth.canCheckBiometrics;
    if (!canAuth) return false;
    
    return localAuth.authenticate(
      localizedReason: 'Authenticate to access AuditFlow',
      options: const AuthenticationOptions(
        stickyAuth: true,
        biometricOnly: true,
      ),
    );
  }
}
```

---

## **8. UI Components**

```dart
// lib/presentation/screens/voucher_entry_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

class VoucherEntryScreen extends ConsumerStatefulWidget {
  final String voucherType;
  
  const VoucherEntryScreen({required this.voucherType, super.key});
  
  @override
  ConsumerState<VoucherEntryScreen> createState() => _VoucherEntryScreenState();
}

class _VoucherEntryScreenState extends ConsumerState<VoucherEntryScreen> {
  final _formKey = GlobalKey<FormState>();
  final _lines = <VoucherLineEntry>[];
  DateTime _voucherDate = DateTime.now();
  String? _narration;
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('${widget.voucherType} Entry'),
        actions: [
          IconButton(
            icon: const Icon(Icons.save),
            onPressed: _saveVoucher,
          ),
        ],
      ),
      body: Form(
        key: _formKey,
        child: ListView(
          padding: const EdgeInsets.all(16),
          children: [
            // Date Picker
            ListTile(
              title: const Text('Date'),
              subtitle: Text(DateFormat('dd-MM-yyyy').format(_voucherDate)),
              trailing: const Icon(Icons.calendar_today),
              onTap: _selectDate,
            ),
            
            const Divider(),
            
            // Voucher Lines
            ..._lines.asMap().entries.map((entry) => 
              _VoucherLineCard(
                line: entry.value,
                index: entry.key,
                onRemove: () => _removeLine(entry.key),
                onEdit: () => _editLine(entry.key),
              ),
            ),
            
            // Add Line Button
            OutlinedButton.icon(
              onPressed: _addLine,
              icon: const Icon(Icons.add),
              label: const Text('Add Entry'),
            ),
            
            const Divider(),
            
            // Totals
            _TotalsCard(lines: _lines),
            
            const SizedBox(height: 16),
            
            // Narration
            TextFormField(
              decoration: const InputDecoration(
                labelText: 'Narration',
                border: OutlineInputBorder(),
              ),
              maxLines: 2,
              onChanged: (v) => _narration = v,
            ),
          ],
        ),
      ),
    );
  }
  
  Future<void> _selectDate() async {
    final picked = await showDatePicker(
      context: context,
      initialDate: _voucherDate,
      firstDate: DateTime(2020),
      lastDate: DateTime.now(),
    );
    if (picked != null) {
      setState(() => _voucherDate = picked);
    }
  }
  
  Future<void> _addLine() async {
    final result = await showModalBottomSheet<VoucherLineEntry>(
      context: context,
      isScrollControlled: true,
      builder: (context) => const VoucherLineEntrySheet(),
    );
    
    if (result != null) {
      setState(() => _lines.add(result));
    }
  }
  
  void _removeLine(int index) {
    setState(() => _lines.removeAt(index));
  }
  
  Future<void> _saveVoucher() async {
    if (!_formKey.currentState!.validate()) return;
    
    // Validate debit = credit
    final totalDebit = _lines.fold<double>(0, (sum, l) => sum + l.debit);
    final totalCredit = _lines.fold<double>(0, (sum, l) => sum + l.credit);
    
    if ((totalDebit - totalCredit).abs() > 0.01) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Debit and Credit must be equal')),
      );
      return;
    }
    
    try {
      await ref.read(voucherNotifierProvider.notifier).saveVoucher(
        VoucherEntry(
          voucherType: widget.voucherType,
          voucherDate: _voucherDate,
          narration: _narration,
          lines: _lines,
        ),
      );
      
      if (mounted) {
        Navigator.pop(context);
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Voucher saved successfully')),
        );
      }
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: $e')),
      );
    }
  }
}

// lib/presentation/screens/dashboard_screen.dart
class DashboardScreen extends ConsumerWidget {
  const DashboardScreen({super.key});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final dashboardAsync = ref.watch(dashboardNotifierProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: const Text('Dashboard'),
        actions: [
          // Sync status indicator
          Consumer(
            builder: (context, ref, _) {
              final syncStatus = ref.watch(syncStatusProvider);
              return IconButton(
                icon: Icon(
                  syncStatus.isSyncing ? Icons.sync : Icons.cloud_done,
                  color: syncStatus.hasPendingChanges ? Colors.orange : Colors.green,
                ),
                onPressed: () => ref.read(syncEngineProvider).fullSync(),
              );
            },
          ),
        ],
      ),
      body: dashboardAsync.when(
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (e, st) => Center(child: Text('Error: $e')),
        data: (dashboard) => RefreshIndicator(
          onRefresh: () => ref.read(dashboardNotifierProvider.notifier).refresh(),
          child: ListView(
            padding: const EdgeInsets.all(16),
            children: [
              // KPI Cards
              GridView.count(
                shrinkWrap: true,
                physics: const NeverScrollableScrollPhysics(),
                crossAxisCount: 2,
                mainAxisSpacing: 12,
                crossAxisSpacing: 12,
                childAspectRatio: 1.5,
                children: [
                  _KPICard(
                    title: 'Today\'s Sales',
                    value: '₹${dashboard.todaySales.toIndianFormat()}',
                    icon: Icons.trending_up,
                    color: Colors.green,
                  ),
                  _KPICard(
                    title: 'Receivables',
                    value: '₹${dashboard.receivables.toIndianFormat()}',
                    icon: Icons.account_balance_wallet,
                    color: Colors.orange,
                  ),
                  _KPICard(
                    title: 'Payables',
                    value: '₹${dashboard.payables.toIndianFormat()}',
                    icon: Icons.payment,
                    color: Colors.red,
                  ),
                  _KPICard(
                    title: 'Cash in Hand',
                    value: '₹${dashboard.cashBalance.toIndianFormat()}',
                    icon: Icons.money,
                    color: Colors.blue,
                  ),
                ],
              ),
              
              const SizedBox(height: 24),
              
              // Quick Actions
              const Text('Quick Actions', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
              const SizedBox(height: 12),
              
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: [
                  _QuickAction(
                    icon: Icons.receipt_long,
                    label: 'Sales',
                    onTap: () => Navigator.pushNamed(context, '/voucher/Sales'),
                  ),
                  _QuickAction(
                    icon: Icons.shopping_cart,
                    label: 'Purchase',
                    onTap: () => Navigator.pushNamed(context, '/voucher/Purchase'),
                  ),
                  _QuickAction(
                    icon: Icons.arrow_downward,
                    label: 'Receipt',
                    onTap: () => Navigator.pushNamed(context, '/voucher/Receipt'),
                  ),
                  _QuickAction(
                    icon: Icons.arrow_upward,
                    label: 'Payment',
                    onTap: () => Navigator.pushNamed(context, '/voucher/Payment'),
                  ),
                ],
              ),
              
              const SizedBox(height: 24),
              
              // Recent Transactions
              const Text('Recent Transactions', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
              ...dashboard.recentVouchers.map((v) => _TransactionTile(voucher: v)),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## **9. API Client**

```dart
// lib/data/remote/api_client.dart
import 'package:dio/dio.dart';
import 'package:retrofit/retrofit.dart';

part 'api_client.g.dart';

@RestApi()
abstract class ApiClient {
  factory ApiClient(Dio dio, {String baseUrl}) = _ApiClient;
  
  // Auth
  @POST('/api/v1/auth/login')
  Future<AuthResponse> login(@Body() LoginRequest request);
  
  @POST('/api/v1/auth/refresh')
  Future<AuthResponse> refreshToken(@Body() RefreshTokenRequest request);
  
  // Vouchers
  @GET('/api/v1/vouchers')
  Future<PagedResponse<VoucherDto>> getVouchers(
    @Query('fromDate') String? fromDate,
    @Query('toDate') String? toDate,
    @Query('type') String? type,
    @Query('page') int page,
    @Query('pageSize') int pageSize,
  );
  
  @POST('/api/v1/vouchers')
  Future<VoucherDto> createVoucher(@Body() CreateVoucherRequest request);
  
  // Sync
  @GET('/api/v1/sync/changes')
  Future<SyncChangesResponse> getChanges(@Query('since') String since);
  
  @POST('/api/v1/sync/push')
  Future<SyncPushResponse> pushChanges(@Body() List<SyncItem> changes);
  
  // Ledgers
  @GET('/api/v1/ledgers')
  Future<List<LedgerDto>> getLedgers(@Query('search') String? search);
  
  @GET('/api/v1/ledgers/{id}/statement')
  Future<LedgerStatementDto> getLedgerStatement(
    @Path('id') String ledgerId,
    @Query('fromDate') String fromDate,
    @Query('toDate') String toDate,
  );
  
  // Dashboard
  @GET('/api/v1/reports/dashboard')
  Future<DashboardDto> getDashboard(
    @Query('fromDate') String fromDate,
    @Query('toDate') String toDate,
  );
  
  // Items
  @GET('/api/v1/inventory/items')
  Future<PagedResponse<ItemDto>> getItems(
    @Query('search') String? search,
    @Query('page') int page,
    @Query('pageSize') int pageSize,
  );
  
  @GET('/api/v1/inventory/stock-summary')
  Future<StockSummaryDto> getStockSummary();
}

// Dio configuration with interceptors
Dio createDio(String baseUrl, SecureStorage storage) {
  final dio = Dio(BaseOptions(
    baseUrl: baseUrl,
    connectTimeout: const Duration(seconds: 30),
    receiveTimeout: const Duration(seconds: 30),
  ));
  
  // Auth interceptor
  dio.interceptors.add(InterceptorsWrapper(
    onRequest: (options, handler) async {
      final token = await storage.read(key: 'auth_token');
      if (token != null) {
        options.headers['Authorization'] = 'Bearer $token';
      }
      handler.next(options);
    },
    onError: (error, handler) async {
      if (error.response?.statusCode == 401) {
        // Try refresh token
        try {
          final newToken = await _refreshToken(dio, storage);
          error.requestOptions.headers['Authorization'] = 'Bearer $newToken';
          final response = await dio.fetch(error.requestOptions);
          handler.resolve(response);
        } catch (e) {
          // Force logout
          await storage.deleteAll();
          handler.next(error);
        }
      } else {
        handler.next(error);
      }
    },
  ));
  
  // Logging interceptor
  dio.interceptors.add(LogInterceptor(
    requestBody: true,
    responseBody: true,
  ));
  
  return dio;
}
```

---

## **10. Project Structure**

```
lib/
├── main.dart
├── app.dart
├── core/
│   ├── constants/
│   ├── errors/
│   ├── extensions/
│   ├── sync/
│   │   ├── sync_engine.dart
│   │   └── conflict_resolver.dart
│   └── utils/
├── data/
│   ├── local/
│   │   ├── database.dart
│   │   └── database.g.dart
│   ├── remote/
│   │   ├── api_client.dart
│   │   └── api_client.g.dart
│   └── repositories/
│       ├── voucher_repository.dart
│       ├── ledger_repository.dart
│       └── auth_repository.dart
├── domain/
│   ├── entities/
│   │   ├── voucher.dart
│   │   ├── ledger.dart
│   │   └── item.dart
│   ├── repositories/
│   └── usecases/
└── presentation/
    ├── providers/
    │   ├── auth_provider.dart
    │   ├── voucher_provider.dart
    │   └── dashboard_provider.dart
    ├── screens/
    │   ├── login_screen.dart
    │   ├── dashboard_screen.dart
    │   ├── voucher_entry_screen.dart
    │   └── ledger_statement_screen.dart
    ├── widgets/
    │   ├── kpi_card.dart
    │   ├── voucher_line_card.dart
    │   └── sync_status_indicator.dart
    └── theme/
        └── app_theme.dart
```

---

## **11. Acceptance Criteria**

| Feature | Criteria |
|---------|----------|
| Offline Mode | App functional without internet |
| Sync | Auto-sync when online within 30s |
| Conflict | Handle conflicts gracefully with user notification |
| Biometric | Support Face ID / Fingerprint |
| Voucher Entry | Create Receipt/Payment in < 30s |
| Dashboard | Load in < 2s (from local) |
| App Size | < 30 MB (Android APK) |

---

## **12. Definition of Done**

- [ ] Login with email/password
- [ ] Biometric authentication
- [ ] Company selection
- [ ] Dashboard with KPIs
- [ ] Receipt voucher entry
- [ ] Payment voucher entry
- [ ] View ledger statement
- [ ] View party outstanding
- [ ] View stock summary
- [ ] Offline data storage (Drift)
- [ ] Sync engine with conflict resolution
- [ ] Push notifications for sync conflicts
- [ ] Android build and Play Store ready
- [ ] iOS build and App Store ready
- [ ] Crash reporting (Firebase Crashlytics)
- [ ] Analytics (Firebase Analytics)
