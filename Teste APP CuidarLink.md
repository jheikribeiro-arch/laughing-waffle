# Performance - CuidarLink

## Estratégias de Performance

CuidarLink implementa múltiplas estratégias para garantir alta performance, tempo de carregamento rápido e experiência de usuário fluida.

## Estratégias de Performance

### 1. Paginação e Lazy Loading

#### Busca Paginada de Cuidadores
```dart
class CaregiverRepository {
  Future<List<CaregiverProfile>> searchCaregivers({
    String? city,
    Rating? minRating,
    Money? maxHourlyRate,
    int limit = 20,
    String? lastDocumentId,
  }) async {
    var query = _firestore.collection('caregiver_profiles')
      .where('isApproved', isEqualTo: true);

    // Aplicar filtros
    if (city != null) {
      query = query.where('city', isEqualTo: city);
    }
    if (minRating != null) {
      query = query.where('averageRating', isGreaterThanOrEqualTo: minRating.value);
    }
    if (maxHourlyRate != null) {
      query = query.where('hourlyRate', isLessThanOrEqualTo: maxHourlyRate.amount);
    }

    // Ordenar por rating e aplicar limite
    query = query.orderBy('averageRating', descending: true)
      .limit(limit);

    // Paginação
    if (lastDocumentId != null) {
      final lastDoc = await _firestore
        .collection('caregiver_profiles')
        .doc(lastDocumentId)
        .get();
      query = query.startAfterDocument(lastDoc);
    }

    final snapshot = await query.get();
    return snapshot.docs.map((doc) => 
      CaregiverProfile.fromMap(doc.data())
    ).toList();
  }
}
```

#### Lazy Loading de Imagens
```dart
class CaregiverCard extends StatelessWidget {
  final CaregiverProfile profile;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Column(
        children: [
          // Lazy loading com cache
          CachedNetworkImage(
            imageUrl: profile.photoUrl ?? 'default_avatar.png',
            placeholder: (context, url) => Shimmer.fromColors(
              baseColor: Colors.grey[300]!,
              highlightColor: Colors.grey[100]!,
              child: Container(
                height: 120,
                width: double.infinity,
                color: Colors.white,
              ),
            ),
            errorWidget: (context, url, error) => Icon(Icons.error),
            fit: BoxFit.cover,
            width: double.infinity,
            height: 120,
          ),
          // Conteúdo do card
        ],
      ),
    );
  }
}
```

### 2. Estratégias de Cache

#### Cache Local com SharedPreferences
```dart
class LocalCache {
  final SharedPreferences _prefs;
  final Duration _cacheDuration = Duration(hours: 1);

  Future<void> set<T>(String key, T value) async {
    final data = {
      'value': value,
      'timestamp': DateTime.now().toIso8601String()
    };
    await _prefs.setString(key, jsonEncode(data));
  }

  Future<T?> get<T>(String key) async {
    final data = _prefs.getString(key);
    if (data == null) return null;

    final parsed = jsonDecode(data);
    final timestamp = DateTime.parse(parsed['timestamp']);
    
    if (DateTime.now().difference(timestamp) > _cacheDuration) {
      await _prefs.remove(key);
      return null;
    }

    return parsed['value'] as T;
  }

  Future<void> clear() async {
    await _prefs.clear();
  }
}
```

#### Cache em Memória com Riverpod
```dart
final caregiverCacheProvider = StateProvider<Map<String, CaregiverProfile>>((ref) {
  return {};
});

final caregiverListProvider = FutureProvider<List<CaregiverProfile>>((ref) async {
  final cache = ref.watch(caregiverCacheProvider);
  
  // Verificar cache primeiro
  if (cache.isNotEmpty) {
    return cache.values.toList();
  }

  // Buscar do Firestore
  final repository = ref.watch(caregiverRepositoryProvider);
  final caregivers = await repository.searchCaregivers();
  
  // Atualizar cache
  final cacheMap = <String, CaregiverProfile>{};
  for (final caregiver in caregivers) {
    cacheMap[caregiver.userId] = caregiver;
  }
  ref.read(caregiverCacheProvider.notifier).state = cacheMap;
  
  return caregivers;
});
```

### 3. Otimização de Consultas Firestore

#### Índices Estratégicos
```javascript
// Índices recomendados para Firestore
// users: role + status
// caregiver_profiles: city, isApproved, hourlyRate
// service_requests: caregiverId + status, contractorId + status

// Consulta otimizada para busca de cuidadores
const query = firestore.collection('caregiver_profiles')
  .where('isApproved', '==', true)
  .where('city', '==', 'São Paulo')
  .where('averageRating', '>=', 4.0)
  .orderBy('averageRating', 'desc')
  .limit(20);
```

#### Batch Operations
```dart
class BatchService {
  final FirebaseFirestore _firestore;

  Future<void> updateMultipleCaregivers(List<String> caregiverIds, Map<String, dynamic> updates) async {
    final batch = _firestore.batch();
    
    for (final caregiverId in caregiverIds) {
      final docRef = _firestore.collection('caregiver_profiles').doc(caregiverId);
      batch.update(docRef, updates);
    }
    
    await batch.commit();
  }

  Future<void> createServiceRequestWithReviews(ServiceRequest request, List<Review> reviews) async {
    final batch = _firestore.batch();
    
    // Criar solicitação de serviço
    final requestRef = _firestore.collection('service_requests').doc();
    batch.set(requestRef, request.toMap());
    
    // Criar avaliações associadas
    for (final review in reviews) {
      final reviewRef = _firestore.collection('reviews').doc();
      batch.set(reviewRef, review.toMap());
    }
    
    await batch.commit();
  }
}
```

### 4. Performance no Flutter

#### Widget Optimization
```dart
class OptimizedCaregiverList extends StatelessWidget {
  final List<CaregiverProfile> caregivers;

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: caregivers.length,
      itemBuilder: (context, index) {
        // Usar const widgets quando possível
        return const CaregiverListItem(
          // Parâmetros aqui
        );
      },
      // Evitar rebuilds desnecessários
      cacheExtent: 500.0,
      // Use shrinkWrap: false para melhor performance em listas longas
      shrinkWrap: false,
    );
  }
}

class CaregiverListItem extends StatelessWidget {
  final CaregiverProfile profile;

  const CaregiverListItem({Key? key, required this.profile}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    // Memoização de widgets complexos
    return MemoizedWidget(
      condition: profile.hashCode,
      child: _buildContent(),
    );
  }

  Widget _buildContent() {
    return ListTile(
      leading: CircleAvatar(
        backgroundImage: NetworkImage(profile.photoUrl ?? 'default.png'),
      ),
      title: Text(profile.name),
      subtitle: Text('${profile.city} • R\$ ${profile.hourlyRate}/h'),
      trailing: RatingWidget(rating: profile.averageRating),
    );
  }
}
```

#### Image Optimization
```dart
class OptimizedImage extends StatelessWidget {
  final String imageUrl;
  final double width;
  final double height;

  const OptimizedImage({
    Key? key,
    required this.imageUrl,
    required this.width,
    required this.height,
  }) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Image.network(
      imageUrl,
      width: width,
      height: height,
      fit: BoxFit.cover,
      // Carregamento progressivo
      loadingBuilder: (context, child, loadingProgress) {
        if (loadingProgress == null) return child;
        return Shimmer.fromColors(
          baseColor: Colors.grey[300]!,
          highlightColor: Colors.grey[100]!,
          child: Container(
            width: width,
            height: height,
            color: Colors.white,
          ),
        );
      },
      // Tratamento de erros
      errorBuilder: (context, error, stackTrace) {
        return Icon(Icons.error, size: 40);
      },
    );
  }
}
```

### 5. Estratégias de Redução de Bandwidth

#### Data Compression
```dart
class DataCompressionService {
  // Comprimir imagens antes do upload
  Future<Uint8List> compressImage(File imageFile) async {
    final result = await FlutterImageCompress.compressWithFile(
      imageFile.absolute.path,
      minWidth: 400,
      minHeight: 400,
      quality: 85,
      rotate: 0,
    );
    return result!;
  }

  // Comprimir JSON responses
  String compressJson(Map<String, dynamic> data) {
    final jsonString = jsonEncode(data);
    final bytes = utf8.encode(jsonString);
    final compressed = GZipEncoder().encode(bytes);
    return base64Encode(compressed!);
  }

  Map<String, dynamic> decompressJson(String compressedData) {
    final bytes = base64Decode(compressedData);
    final decompressed = GZipDecoder().decodeBytes(bytes);
    final jsonString = utf8.decode(decompressed);
    return jsonDecode(jsonString);
  }
}
```

#### Selective Data Loading
```dart
class SelectiveDataService {
  // Carregar apenas dados essenciais
  Future<CaregiverProfileSummary> getCaregiverSummary(String userId) async {
    final doc = await _firestore.collection('caregiver_profiles')
      .doc(userId)
      .get();
    
    return CaregiverProfileSummary.fromMap(doc.data()!);
  }

  // Carregar dados completos apenas quando necessário
  Future<CaregiverProfile> getCaregiverFullProfile(String userId) async {
    final summary = await getCaregiverSummary(userId);
    
    // Carregar dados adicionais
    final reviewsFuture = getCaregiverReviews(userId);
    final certificationsFuture = getCaregiverCertifications(userId);
    
    final results = await Future.wait([
      reviewsFuture,
      certificationsFuture,
    ]);
    
    return CaregiverProfile(
      summary: summary,
      reviews: results[0],
      certifications: results[1],
    );
  }
}
```

### 6. Performance Monitoring

#### Firebase Performance Monitoring
```dart
class PerformanceMonitoring {
  final FirebasePerformance _performance;

  void startTrace(String traceName) {
    final trace = _performance.newTrace(traceName);
    trace.start();
  }

  void stopTrace(String traceName) {
    final trace = _performance.getTrace(traceName);
    trace.stop();
  }

  void recordMetric(String traceName, String metricName, int value) {
    final trace = _performance.getTrace(traceName);
    trace.incrementMetric(metricName, value);
  }

  void logCustomEvent(String eventName, Map<String, dynamic> parameters) {
    FirebaseAnalytics.instance.logEvent(
      name: eventName,
      parameters: parameters,
    );
  }
}
```

#### Custom Performance Metrics
```dart
class CustomMetrics {
  static final Stopwatch _appStartupTimer = Stopwatch();
  static final Stopwatch _apiCallTimer = Stopwatch();

  static void startAppStartup() {
    _appStartupTimer.start();
  }

  static void endAppStartup() {
    _appStartupTimer.stop();
    final duration = _appStartupTimer.elapsedMilliseconds;
    
    // Log de performance
    FirebasePerformance.instance
      .newTrace('app_startup')
      .incrementMetric('duration_ms', duration);
  }

  static Future<T> measureApiCall<T>(Future<T> Function() apiCall) async {
    _apiCallTimer.start();
    
    try {
      final result = await apiCall();
      _apiCallTimer.stop();
      
      FirebasePerformance.instance
        .newTrace('api_call')
        .incrementMetric('success_duration_ms', _apiCallTimer.elapsedMilliseconds);
      
      return result;
    } catch (e) {
      _apiCallTimer.stop();
      
      FirebasePerformance.instance
        .newTrace('api_call')
        .incrementMetric('error_duration_ms', _apiCallTimer.elapsedMilliseconds);
      
      rethrow;
    }
  }
}
```

## Estratégias de Otimização Específica

### 1. Database Optimization

#### Query Optimization
```javascript
// Índices recomendados
// users: [role, status]
// caregiver_profiles: [city, isApproved, averageRating]
// service_requests: [contractorId, status], [caregiverId, status]

// Consultas otimizadas
exports.getAvailableCaregivers = functions.https.onCall(async (data, context) => {
  const { city, minRating, maxHourlyRate } = data;
  
  let query = admin.firestore()
    .collection('caregiver_profiles')
    .where('isApproved', '==', true);

  if (city) query = query.where('city', '==', city);
  if (minRating) query = query.where('averageRating', '>=', minRating);
  if (maxHourlyRate) query = query.where('hourlyRate', '<=', maxHourlyRate);

  query = query.orderBy('averageRating', 'desc').limit(50);

  const snapshot = await query.get();
  return snapshot.docs.map(doc => doc.data());
});
```

#### Data Structure Optimization
```javascript
// Estrutura otimizada para consultas frequentes
{
  // Documento do cuidador
  userId: "user_123",
  basicInfo: {
    name: "João Silva",
    city: "São Paulo",
    hourlyRate: 50.0,
    isApproved: true
  },
  ratings: {
    averageRating: 4.8,
    totalReviews: 150,
    recentReviews: [/* últimos 5 reviews */]
  },
  availability: {
    schedule: { /* horários disponíveis */ },
    lastUpdated: "2024-01-01T00:00:00Z"
  }
}
```

### 2. Network Optimization

#### Connection Management
```dart
class NetworkManager {
  static final Dio _dio = Dio();
  static final Connectivity _connectivity = Connectivity();

  static Future<void> init() async {
    // Configurar timeouts
    _dio.options.connectTimeout = const Duration(seconds: 10);
    _dio.options.receiveTimeout = const Duration(seconds: 15);
    _dio.options.sendTimeout = const Duration(seconds: 10);

    // Configurar retry
    _dio.interceptors.add(RetryInterceptor(
      dio: _dio,
      retries: 3,
      retryInterval: const Duration(seconds: 1),
    ));
  }

  static Future<bool> checkConnection() async {
    final result = await _connectivity.checkConnectivity();
    return result != ConnectivityResult.none;
  }
}
```

#### Request Batching
```dart
class RequestBatcher {
  final List<Future> _pendingRequests = [];
  final Duration _batchWindow = const Duration(milliseconds: 100);

  Future<T> batchRequest<T>(Future<T> Function() request) async {
    _pendingRequests.add(request());
    
    await Future.delayed(_batchWindow);
    
    final results = await Future.wait(_pendingRequests);
    _pendingRequests.clear();
    
    return results.first as T;
  }
}
```

## Performance Benchmarks

### Metas de Performance

#### Tempo de Carregamento
- **Splash Screen:** < 2s
- **Login:** < 3s
- **Busca de Cuidadores:** < 2s
- **Perfil de Cuidador:** < 1.5s

#### Tempo de Resposta
- **API Calls:** < 1s (95% das requisições)
- **Busca:** < 500ms
- **Atualização de Dados:** < 1s

#### Uso de Recursos
- **Memória:** < 100MB (app em foreground)
- **Bateria:** < 5% de drenagem por hora de uso
- **Armazenamento:** < 50MB de cache

### Monitoramento de Performance

#### KPIs Monitorados
- **App Startup Time**
- **Screen Render Time**
- **API Response Time**
- **Image Load Time**
- **Memory Usage**
- **Battery Consumption**

#### Alertas de Performance
```dart
class PerformanceAlerts {
  static void checkPerformanceMetrics() {
    // Verificar métricas críticas
    if (appStartupTime > 3000) {
      FirebaseCrashlytics.instance.recordError(
        Exception('App startup too slow'),
        null,
        reason: 'Performance issue',
      );
    }
    
    if (apiResponseTime > 2000) {
      FirebaseCrashlytics.instance.recordError(
        Exception('API response too slow'),
        null,
        reason: 'Performance issue',
      );
    }
  }
}
```

## Conclusão

As estratégias de performance do CuidarLink garantem:

- **Experiência de usuário fluida** com tempos de carregamento rápidos
- **Uso eficiente de recursos** do dispositivo
- **Otimização de banda** para usuários com conexões limitadas
- **Monitoramento contínuo** para identificar e resolver problemas rapidamente

A combinação de técnicas de cache, paginação inteligente, otimização de consultas e monitoramento de performance garante que o aplicativo mantenha alta performance mesmo com crescimento de usuários e dados.