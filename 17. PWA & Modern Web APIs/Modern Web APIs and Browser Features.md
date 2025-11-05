# Modern Web APIs and Browser Features

Modern web APIs provide powerful capabilities that enable rich, native-like experiences in web applications. This guide covers essential browser APIs, integration patterns, and practical implementations for building advanced web applications.

## Storage and Data Management APIs

### 1. Advanced Storage Solutions
```typescript
// IndexedDB Wrapper for Complex Data Storage
interface IndexedDBConfig {
  databaseName: string;
  version: number;
  stores: Array<{
    name: string;
    keyPath?: string;
    autoIncrement?: boolean;
    indexes?: Array<{
      name: string;
      keyPath: string;
      unique?: boolean;
    }>;
  }>;
}

class IndexedDBManager {
  private db: IDBDatabase | null = null;
  private config: IndexedDBConfig;
  private isReady = false;

  constructor(config: IndexedDBConfig) {
    this.config = config;
  }

  // Initialize database
  async initialize(): Promise<void> {
    return new Promise((resolve, reject) => {
      if (!window.indexedDB) {
        reject(new Error('IndexedDB not supported'));
        return;
      }

      const request = indexedDB.open(this.config.databaseName, this.config.version);

      request.onerror = () => {
        reject(new Error('IndexedDB initialization failed'));
      };

      request.onsuccess = () => {
        this.db = request.result;
        this.isReady = true;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;
        this.upgradeDatabase(db);
      };
    });
  }

  private upgradeDatabase(db: IDBDatabase): void {
    // Create object stores
    this.config.stores.forEach(storeConfig => {
      let objectStore: IDBObjectStore;

      if (db.objectStoreNames.contains(storeConfig.name)) {
        // Store already exists, we might need to handle version upgrades
        return;
      }

      objectStore = db.createObjectStore(storeConfig.name, {
        keyPath: storeConfig.keyPath,
        autoIncrement: storeConfig.autoIncrement || false,
      });

      // Create indexes
      if (storeConfig.indexes) {
        storeConfig.indexes.forEach(indexConfig => {
          objectStore.createIndex(
            indexConfig.name,
            indexConfig.keyPath,
            { unique: indexConfig.unique || false }
          );
        });
      }
    });
  }

  // Generic CRUD operations
  async create<T>(storeName: string, data: T): Promise<IDBValidKey> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.add(data);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async read<T>(storeName: string, key: IDBValidKey): Promise<T | undefined> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.get(key);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async update<T>(storeName: string, data: T): Promise<IDBValidKey> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.put(data);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async delete(storeName: string, key: IDBValidKey): Promise<void> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.delete(key);

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  // Query operations
  async getAll<T>(storeName: string): Promise<T[]> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.getAll();

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async query<T>(
    storeName: string,
    indexName: string,
    query: IDBValidKey | IDBKeyRange
  ): Promise<T[]> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const index = store.index(indexName);
      const request = index.getAll(query);

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  async count(storeName: string): Promise<number> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const request = store.count();

      request.onsuccess = () => resolve(request.result);
      request.onerror = () => reject(request.error);
    });
  }

  // Cursor-based iteration for large datasets
  async iterate<T>(
    storeName: string,
    callback: (value: T, key: IDBValidKey) => boolean | void,
    indexName?: string
  ): Promise<void> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readonly');
      const store = transaction.objectStore(storeName);
      const source = indexName ? store.index(indexName) : store;
      const request = source.openCursor();

      request.onsuccess = () => {
        const cursor = request.result;
        
        if (cursor) {
          const shouldContinue = callback(cursor.value, cursor.key);
          
          if (shouldContinue !== false) {
            cursor.continue();
          } else {
            resolve();
          }
        } else {
          resolve();
        }
      };

      request.onerror = () => reject(request.error);
    });
  }

  // Bulk operations
  async bulkCreate<T>(storeName: string, items: T[]): Promise<IDBValidKey[]> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const results: IDBValidKey[] = [];
      let completed = 0;

      const handleComplete = () => {
        completed++;
        if (completed === items.length) {
          resolve(results);
        }
      };

      items.forEach((item, index) => {
        const request = store.add(item);
        
        request.onsuccess = () => {
          results[index] = request.result;
          handleComplete();
        };
        
        request.onerror = () => {
          reject(request.error);
        };
      });

      if (items.length === 0) {
        resolve([]);
      }
    });
  }

  // Clear store
  async clear(storeName: string): Promise<void> {
    this.ensureReady();
    
    return new Promise((resolve, reject) => {
      const transaction = this.db!.transaction([storeName], 'readwrite');
      const store = transaction.objectStore(storeName);
      const request = store.clear();

      request.onsuccess = () => resolve();
      request.onerror = () => reject(request.error);
    });
  }

  private ensureReady(): void {
    if (!this.isReady || !this.db) {
      throw new Error('IndexedDB not initialized');
    }
  }

  // Close database connection
  close(): void {
    if (this.db) {
      this.db.close();
      this.db = null;
      this.isReady = false;
    }
  }
}

// Enhanced Storage Manager with multiple storage types
interface StorageItem<T = any> {
  value: T;
  timestamp: number;
  expiry?: number;
  version?: string;
}

class EnhancedStorageManager {
  private indexedDB: IndexedDBManager;
  private storageQuota: StorageQuota | null = null;

  constructor(dbConfig: IndexedDBConfig) {
    this.indexedDB = new IndexedDBManager(dbConfig);
    this.initializeStorageQuota();
  }

  async initialize(): Promise<void> {
    await this.indexedDB.initialize();
  }

  private async initializeStorageQuota(): Promise<void> {
    if ('storage' in navigator && 'estimate' in navigator.storage) {
      try {
        const estimate = await navigator.storage.estimate();
        this.storageQuota = {
          quota: estimate.quota || 0,
          usage: estimate.usage || 0,
          usagePercentage: estimate.quota ? (estimate.usage || 0) / estimate.quota * 100 : 0,
        };
      } catch (error) {
        console.warn('Storage quota estimation failed:', error);
      }
    }
  }

  // Local Storage with expiry and versioning
  setLocalStorage<T>(key: string, value: T, expiryMinutes?: number, version?: string): void {
    const item: StorageItem<T> = {
      value,
      timestamp: Date.now(),
      expiry: expiryMinutes ? Date.now() + (expiryMinutes * 60 * 1000) : undefined,
      version,
    };

    try {
      localStorage.setItem(key, JSON.stringify(item));
    } catch (error) {
      console.error('LocalStorage write failed:', error);
      // Attempt cleanup and retry
      this.cleanupExpiredLocalStorage();
      
      try {
        localStorage.setItem(key, JSON.stringify(item));
      } catch (retryError) {
        throw new Error('LocalStorage quota exceeded');
      }
    }
  }

  getLocalStorage<T>(key: string, expectedVersion?: string): T | null {
    try {
      const itemStr = localStorage.getItem(key);
      
      if (!itemStr) {
        return null;
      }

      const item: StorageItem<T> = JSON.parse(itemStr);

      // Check expiry
      if (item.expiry && Date.now() > item.expiry) {
        localStorage.removeItem(key);
        return null;
      }

      // Check version
      if (expectedVersion && item.version !== expectedVersion) {
        localStorage.removeItem(key);
        return null;
      }

      return item.value;
    } catch (error) {
      console.error('LocalStorage read failed:', error);
      return null;
    }
  }

  // Session Storage with the same enhancements
  setSessionStorage<T>(key: string, value: T, version?: string): void {
    const item: StorageItem<T> = {
      value,
      timestamp: Date.now(),
      version,
    };

    try {
      sessionStorage.setItem(key, JSON.stringify(item));
    } catch (error) {
      console.error('SessionStorage write failed:', error);
      throw new Error('SessionStorage quota exceeded');
    }
  }

  getSessionStorage<T>(key: string, expectedVersion?: string): T | null {
    try {
      const itemStr = sessionStorage.getItem(key);
      
      if (!itemStr) {
        return null;
      }

      const item: StorageItem<T> = JSON.parse(itemStr);

      // Check version
      if (expectedVersion && item.version !== expectedVersion) {
        sessionStorage.removeItem(key);
        return null;
      }

      return item.value;
    } catch (error) {
      console.error('SessionStorage read failed:', error);
      return null;
    }
  }

  // IndexedDB operations
  async setIndexedDB<T>(storeName: string, key: IDBValidKey, value: T): Promise<void> {
    await this.indexedDB.update(storeName, { id: key, ...value });
  }

  async getIndexedDB<T>(storeName: string, key: IDBValidKey): Promise<T | undefined> {
    return this.indexedDB.read<T>(storeName, key);
  }

  // Storage cleanup utilities
  cleanupExpiredLocalStorage(): void {
    const keys = Object.keys(localStorage);
    
    keys.forEach(key => {
      try {
        const itemStr = localStorage.getItem(key);
        if (itemStr) {
          const item: StorageItem = JSON.parse(itemStr);
          
          if (item.expiry && Date.now() > item.expiry) {
            localStorage.removeItem(key);
          }
        }
      } catch (error) {
        // Invalid JSON, remove the item
        localStorage.removeItem(key);
      }
    });
  }

  // Storage usage analysis
  async getStorageUsage(): Promise<{
    localStorage: number;
    sessionStorage: number;
    indexedDB: number;
    total: number;
    quota: number;
    available: number;
  }> {
    const localStorageSize = this.calculateStorageSize(localStorage);
    const sessionStorageSize = this.calculateStorageSize(sessionStorage);
    
    let indexedDBSize = 0;
    if (this.storageQuota) {
      await this.initializeStorageQuota();
      indexedDBSize = this.storageQuota.usage;
    }

    const total = localStorageSize + sessionStorageSize + indexedDBSize;
    const quota = this.storageQuota?.quota || 0;
    const available = quota - total;

    return {
      localStorage: localStorageSize,
      sessionStorage: sessionStorageSize,
      indexedDB: indexedDBSize,
      total,
      quota,
      available,
    };
  }

  private calculateStorageSize(storage: Storage): number {
    let total = 0;
    
    for (let i = 0; i < storage.length; i++) {
      const key = storage.key(i);
      if (key) {
        const value = storage.getItem(key);
        if (value) {
          total += key.length + value.length;
        }
      }
    }
    
    return total;
  }

  // Request persistent storage
  async requestPersistentStorage(): Promise<boolean> {
    if ('storage' in navigator && 'persist' in navigator.storage) {
      try {
        return await navigator.storage.persist();
      } catch (error) {
        console.error('Persistent storage request failed:', error);
        return false;
      }
    }
    
    return false;
  }

  // Check if storage is persistent
  async isPersistent(): Promise<boolean> {
    if ('storage' in navigator && 'persisted' in navigator.storage) {
      try {
        return await navigator.storage.persisted();
      } catch (error) {
        console.error('Persistent storage check failed:', error);
        return false;
      }
    }
    
    return false;
  }
}

interface StorageQuota {
  quota: number;
  usage: number;
  usagePercentage: number;
}

// React hook for enhanced storage
export function useEnhancedStorage(dbConfig: IndexedDBConfig) {
  const [storageManager] = useState(() => new EnhancedStorageManager(dbConfig));
  const [isReady, setIsReady] = useState(false);
  const [storageUsage, setStorageUsage] = useState<any>(null);

  useEffect(() => {
    const initializeStorage = async () => {
      try {
        await storageManager.initialize();
        setIsReady(true);
        
        // Get initial storage usage
        const usage = await storageManager.getStorageUsage();
        setStorageUsage(usage);
      } catch (error) {
        console.error('Storage initialization failed:', error);
      }
    };

    initializeStorage();
  }, [storageManager]);

  const setItem = useCallback(async <T>(
    type: 'localStorage' | 'sessionStorage' | 'indexedDB',
    key: string,
    value: T,
    options?: { expiryMinutes?: number; version?: string; storeName?: string }
  ) => {
    switch (type) {
      case 'localStorage':
        storageManager.setLocalStorage(key, value, options?.expiryMinutes, options?.version);
        break;
      case 'sessionStorage':
        storageManager.setSessionStorage(key, value, options?.version);
        break;
      case 'indexedDB':
        if (!options?.storeName) {
          throw new Error('Store name required for IndexedDB');
        }
        await storageManager.setIndexedDB(options.storeName, key, value);
        break;
    }
  }, [storageManager]);

  const getItem = useCallback(async <T>(
    type: 'localStorage' | 'sessionStorage' | 'indexedDB',
    key: string,
    options?: { version?: string; storeName?: string }
  ): Promise<T | null> => {
    switch (type) {
      case 'localStorage':
        return storageManager.getLocalStorage<T>(key, options?.version);
      case 'sessionStorage':
        return storageManager.getSessionStorage<T>(key, options?.version);
      case 'indexedDB':
        if (!options?.storeName) {
          throw new Error('Store name required for IndexedDB');
        }
        const result = await storageManager.getIndexedDB<T>(options.storeName, key);
        return result || null;
      default:
        return null;
    }
  }, [storageManager]);

  const updateStorageUsage = useCallback(async () => {
    const usage = await storageManager.getStorageUsage();
    setStorageUsage(usage);
  }, [storageManager]);

  return {
    isReady,
    storageUsage,
    setItem,
    getItem,
    updateStorageUsage,
    requestPersistentStorage: () => storageManager.requestPersistentStorage(),
    isPersistent: () => storageManager.isPersistent(),
    cleanupExpired: () => storageManager.cleanupExpiredLocalStorage(),
  };
}
```

## Media and Device APIs

### 1. Camera and Media Capture
```typescript
// Advanced Media Capture Manager
interface MediaCaptureConfig {
  video?: {
    width?: { min?: number; ideal?: number; max?: number };
    height?: { min?: number; ideal?: number; max?: number };
    frameRate?: { min?: number; ideal?: number; max?: number };
    facingMode?: 'user' | 'environment';
    aspectRatio?: number;
  };
  audio?: {
    echoCancellation?: boolean;
    noiseSuppression?: boolean;
    autoGainControl?: boolean;
    sampleRate?: number;
  };
  screen?: {
    video?: boolean;
    audio?: boolean;
    displaySurface?: 'application' | 'browser' | 'monitor' | 'window';
  };
}

class MediaCaptureManager {
  private stream: MediaStream | null = null;
  private mediaRecorder: MediaRecorder | null = null;
  private recordedChunks: Blob[] = [];
  private config: MediaCaptureConfig;

  constructor(config: MediaCaptureConfig = {}) {
    this.config = config;
  }

  // Get user media (camera/microphone)
  async getUserMedia(): Promise<MediaStream> {
    try {
      const constraints: MediaStreamConstraints = {
        video: this.config.video ? {
          width: this.config.video.width,
          height: this.config.video.height,
          frameRate: this.config.video.frameRate,
          facingMode: this.config.video.facingMode,
          aspectRatio: this.config.video.aspectRatio,
        } : false,
        audio: this.config.audio ? {
          echoCancellation: this.config.audio.echoCancellation,
          noiseSuppression: this.config.audio.noiseSuppression,
          autoGainControl: this.config.audio.autoGainControl,
          sampleRate: this.config.audio.sampleRate,
        } : false,
      };

      this.stream = await navigator.mediaDevices.getUserMedia(constraints);
      return this.stream;
    } catch (error) {
      console.error('getUserMedia failed:', error);
      throw new Error(`Media access denied: ${error.message}`);
    }
  }

  // Get screen capture
  async getDisplayMedia(): Promise<MediaStream> {
    try {
      if (!navigator.mediaDevices?.getDisplayMedia) {
        throw new Error('Screen capture not supported');
      }

      const constraints: DisplayMediaStreamConstraints = {
        video: this.config.screen?.video !== false ? {
          displaySurface: this.config.screen?.displaySurface,
        } : false,
        audio: this.config.screen?.audio || false,
      };

      this.stream = await navigator.mediaDevices.getDisplayMedia(constraints);
      return this.stream;
    } catch (error) {
      console.error('getDisplayMedia failed:', error);
      throw new Error(`Screen capture failed: ${error.message}`);
    }
  }

  // Start recording
  async startRecording(options: {
    mimeType?: string;
    videoBitsPerSecond?: number;
    audioBitsPerSecond?: number;
  } = {}): Promise<void> {
    if (!this.stream) {
      throw new Error('No media stream available');
    }

    try {
      const mimeType = options.mimeType || this.getSupportedMimeType();
      
      this.mediaRecorder = new MediaRecorder(this.stream, {
        mimeType,
        videoBitsPerSecond: options.videoBitsPerSecond,
        audioBitsPerSecond: options.audioBitsPerSecond,
      });

      this.recordedChunks = [];

      this.mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) {
          this.recordedChunks.push(event.data);
        }
      };

      this.mediaRecorder.start(1000); // Record in 1-second chunks
    } catch (error) {
      console.error('Recording start failed:', error);
      throw new Error(`Recording failed: ${error.message}`);
    }
  }

  // Stop recording
  async stopRecording(): Promise<Blob> {
    return new Promise((resolve, reject) => {
      if (!this.mediaRecorder) {
        reject(new Error('No active recording'));
        return;
      }

      this.mediaRecorder.onstop = () => {
        const mimeType = this.mediaRecorder?.mimeType || 'video/webm';
        const blob = new Blob(this.recordedChunks, { type: mimeType });
        resolve(blob);
      };

      this.mediaRecorder.stop();
    });
  }

  // Capture still image
  async captureImage(canvas?: HTMLCanvasElement): Promise<{
    blob: Blob;
    dataURL: string;
    canvas: HTMLCanvasElement;
  }> {
    if (!this.stream) {
      throw new Error('No media stream available');
    }

    const videoTrack = this.stream.getVideoTracks()[0];
    if (!videoTrack) {
      throw new Error('No video track available');
    }

    // Create or use provided canvas
    const captureCanvas = canvas || document.createElement('canvas');
    const context = captureCanvas.getContext('2d');
    
    if (!context) {
      throw new Error('Canvas context not available');
    }

    // Create video element for capture
    const video = document.createElement('video');
    video.srcObject = this.stream;
    video.play();

    return new Promise((resolve, reject) => {
      video.onloadedmetadata = () => {
        captureCanvas.width = video.videoWidth;
        captureCanvas.height = video.videoHeight;
        
        context.drawImage(video, 0, 0);
        
        captureCanvas.toBlob((blob) => {
          if (blob) {
            resolve({
              blob,
              dataURL: captureCanvas.toDataURL('image/png'),
              canvas: captureCanvas,
            });
          } else {
            reject(new Error('Image capture failed'));
          }
        }, 'image/png');
      };

      video.onerror = () => {
        reject(new Error('Video loading failed'));
      };
    });
  }

  // Apply filters/effects
  applyVideoFilter(canvas: HTMLCanvasElement, filter: {
    type: 'grayscale' | 'sepia' | 'blur' | 'brightness' | 'contrast';
    value?: number;
  }): void {
    const context = canvas.getContext('2d');
    if (!context) return;

    switch (filter.type) {
      case 'grayscale':
        context.filter = 'grayscale(100%)';
        break;
      case 'sepia':
        context.filter = 'sepia(100%)';
        break;
      case 'blur':
        context.filter = `blur(${filter.value || 2}px)`;
        break;
      case 'brightness':
        context.filter = `brightness(${filter.value || 150}%)`;
        break;
      case 'contrast':
        context.filter = `contrast(${filter.value || 150}%)`;
        break;
    }
  }

  // Get available devices
  async getDevices(): Promise<{
    videoInputs: MediaDeviceInfo[];
    audioInputs: MediaDeviceInfo[];
    audioOutputs: MediaDeviceInfo[];
  }> {
    try {
      const devices = await navigator.mediaDevices.enumerateDevices();
      
      return {
        videoInputs: devices.filter(device => device.kind === 'videoinput'),
        audioInputs: devices.filter(device => device.kind === 'audioinput'),
        audioOutputs: devices.filter(device => device.kind === 'audiooutput'),
      };
    } catch (error) {
      console.error('Device enumeration failed:', error);
      throw new Error('Device access failed');
    }
  }

  // Switch camera
  async switchCamera(deviceId: string): Promise<MediaStream> {
    if (this.stream) {
      this.stream.getTracks().forEach(track => track.stop());
    }

    const updatedConfig = {
      ...this.config,
      video: {
        ...this.config.video,
        deviceId: { exact: deviceId },
      },
    };

    const tempManager = new MediaCaptureManager(updatedConfig);
    this.stream = await tempManager.getUserMedia();
    return this.stream;
  }

  private getSupportedMimeType(): string {
    const mimeTypes = [
      'video/webm;codecs=vp9,opus',
      'video/webm;codecs=vp8,opus',
      'video/webm',
      'video/mp4',
    ];

    for (const mimeType of mimeTypes) {
      if (MediaRecorder.isTypeSupported(mimeType)) {
        return mimeType;
      }
    }

    return 'video/webm';
  }

  // Clean up resources
  stopAllTracks(): void {
    if (this.stream) {
      this.stream.getTracks().forEach(track => track.stop());
      this.stream = null;
    }

    if (this.mediaRecorder && this.mediaRecorder.state !== 'inactive') {
      this.mediaRecorder.stop();
    }
  }

  // Get current stream
  getCurrentStream(): MediaStream | null {
    return this.stream;
  }

  // Check recording state
  get isRecording(): boolean {
    return this.mediaRecorder?.state === 'recording';
  }
}

// React hook for media capture
export function useMediaCapture(config: MediaCaptureConfig = {}) {
  const [mediaManager] = useState(() => new MediaCaptureManager(config));
  const [stream, setStream] = useState<MediaStream | null>(null);
  const [isRecording, setIsRecording] = useState(false);
  const [devices, setDevices] = useState<{
    videoInputs: MediaDeviceInfo[];
    audioInputs: MediaDeviceInfo[];
    audioOutputs: MediaDeviceInfo[];
  } | null>(null);

  useEffect(() => {
    return () => {
      mediaManager.stopAllTracks();
    };
  }, [mediaManager]);

  const startCamera = useCallback(async () => {
    try {
      const mediaStream = await mediaManager.getUserMedia();
      setStream(mediaStream);
      return mediaStream;
    } catch (error) {
      console.error('Camera start failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const startScreenCapture = useCallback(async () => {
    try {
      const mediaStream = await mediaManager.getDisplayMedia();
      setStream(mediaStream);
      return mediaStream;
    } catch (error) {
      console.error('Screen capture start failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const startRecording = useCallback(async (options?: any) => {
    try {
      await mediaManager.startRecording(options);
      setIsRecording(true);
    } catch (error) {
      console.error('Recording start failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const stopRecording = useCallback(async () => {
    try {
      const blob = await mediaManager.stopRecording();
      setIsRecording(false);
      return blob;
    } catch (error) {
      console.error('Recording stop failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const captureImage = useCallback(async (canvas?: HTMLCanvasElement) => {
    return mediaManager.captureImage(canvas);
  }, [mediaManager]);

  const getDeviceList = useCallback(async () => {
    try {
      const deviceList = await mediaManager.getDevices();
      setDevices(deviceList);
      return deviceList;
    } catch (error) {
      console.error('Device enumeration failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const switchCamera = useCallback(async (deviceId: string) => {
    try {
      const newStream = await mediaManager.switchCamera(deviceId);
      setStream(newStream);
      return newStream;
    } catch (error) {
      console.error('Camera switch failed:', error);
      throw error;
    }
  }, [mediaManager]);

  const stopCapture = useCallback(() => {
    mediaManager.stopAllTracks();
    setStream(null);
    setIsRecording(false);
  }, [mediaManager]);

  return {
    stream,
    isRecording,
    devices,
    startCamera,
    startScreenCapture,
    startRecording,
    stopRecording,
    captureImage,
    getDeviceList,
    switchCamera,
    stopCapture,
  };
}
```

### 2. Geolocation and Device Sensors
```typescript
// Advanced Geolocation Manager
interface GeolocationConfig {
  enableHighAccuracy?: boolean;
  timeout?: number;
  maximumAge?: number;
  watchPosition?: boolean;
  fallbackProvider?: 'ip' | 'none';
}

interface LocationData {
  latitude: number;
  longitude: number;
  accuracy: number;
  altitude?: number | null;
  altitudeAccuracy?: number | null;
  heading?: number | null;
  speed?: number | null;
  timestamp: number;
  address?: {
    street?: string;
    city?: string;
    state?: string;
    country?: string;
    postalCode?: string;
  };
}

class GeolocationManager {
  private config: GeolocationConfig;
  private watchId: number | null = null;
  private lastKnownPosition: LocationData | null = null;
  private positionListeners = new Set<(position: LocationData) => void>();
  private errorListeners = new Set<(error: GeolocationPositionError) => void>();

  constructor(config: GeolocationConfig = {}) {
    this.config = {
      enableHighAccuracy: true,
      timeout: 10000,
      maximumAge: 60000,
      watchPosition: false,
      fallbackProvider: 'ip',
      ...config,
    };
  }

  // Get current position
  async getCurrentPosition(): Promise<LocationData> {
    return new Promise((resolve, reject) => {
      if (!navigator.geolocation) {
        reject(new Error('Geolocation not supported'));
        return;
      }

      const options: PositionOptions = {
        enableHighAccuracy: this.config.enableHighAccuracy,
        timeout: this.config.timeout,
        maximumAge: this.config.maximumAge,
      };

      navigator.geolocation.getCurrentPosition(
        async (position) => {
          try {
            const locationData = await this.processPosition(position);
            this.lastKnownPosition = locationData;
            resolve(locationData);
          } catch (error) {
            reject(error);
          }
        },
        async (error) => {
          console.error('Geolocation error:', error);
          
          // Try fallback provider
          if (this.config.fallbackProvider === 'ip') {
            try {
              const fallbackLocation = await this.getLocationFromIP();
              resolve(fallbackLocation);
            } catch (fallbackError) {
              reject(error);
            }
          } else {
            reject(error);
          }
        },
        options
      );
    });
  }

  // Start watching position
  startWatching(): void {
    if (!navigator.geolocation) {
      throw new Error('Geolocation not supported');
    }

    if (this.watchId !== null) {
      return; // Already watching
    }

    const options: PositionOptions = {
      enableHighAccuracy: this.config.enableHighAccuracy,
      timeout: this.config.timeout,
      maximumAge: this.config.maximumAge,
    };

    this.watchId = navigator.geolocation.watchPosition(
      async (position) => {
        try {
          const locationData = await this.processPosition(position);
          this.lastKnownPosition = locationData;
          this.notifyPositionListeners(locationData);
        } catch (error) {
          console.error('Position processing error:', error);
        }
      },
      (error) => {
        this.notifyErrorListeners(error);
      },
      options
    );
  }

  // Stop watching position
  stopWatching(): void {
    if (this.watchId !== null) {
      navigator.geolocation.clearWatch(this.watchId);
      this.watchId = null;
    }
  }

  private async processPosition(position: GeolocationPosition): Promise<LocationData> {
    const locationData: LocationData = {
      latitude: position.coords.latitude,
      longitude: position.coords.longitude,
      accuracy: position.coords.accuracy,
      altitude: position.coords.altitude,
      altitudeAccuracy: position.coords.altitudeAccuracy,
      heading: position.coords.heading,
      speed: position.coords.speed,
      timestamp: position.timestamp,
    };

    // Reverse geocoding to get address
    try {
      const address = await this.reverseGeocode(
        position.coords.latitude,
        position.coords.longitude
      );
      locationData.address = address;
    } catch (error) {
      console.warn('Reverse geocoding failed:', error);
    }

    return locationData;
  }

  private async reverseGeocode(lat: number, lng: number): Promise<LocationData['address']> {
    // Using a free geocoding service (in production, use a proper service)
    try {
      const response = await fetch(
        `https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${lat}&longitude=${lng}&localityLanguage=en`
      );
      
      if (!response.ok) {
        throw new Error('Geocoding request failed');
      }

      const data = await response.json();
      
      return {
        street: data.locality,
        city: data.city,
        state: data.principalSubdivision,
        country: data.countryName,
        postalCode: data.postcode,
      };
    } catch (error) {
      console.error('Reverse geocoding error:', error);
      return {};
    }
  }

  private async getLocationFromIP(): Promise<LocationData> {
    try {
      const response = await fetch('https://ipapi.co/json/');
      
      if (!response.ok) {
        throw new Error('IP geolocation request failed');
      }

      const data = await response.json();
      
      return {
        latitude: data.latitude,
        longitude: data.longitude,
        accuracy: 10000, // IP-based location is less accurate
        timestamp: Date.now(),
        address: {
          city: data.city,
          state: data.region,
          country: data.country_name,
          postalCode: data.postal,
        },
      };
    } catch (error) {
      console.error('IP geolocation error:', error);
      throw new Error('IP geolocation failed');
    }
  }

  // Calculate distance between two points
  calculateDistance(
    lat1: number,
    lng1: number,
    lat2: number,
    lng2: number,
    unit: 'km' | 'miles' = 'km'
  ): number {
    const R = unit === 'km' ? 6371 : 3959; // Earth's radius
    const dLat = this.toRadians(lat2 - lat1);
    const dLng = this.toRadians(lng2 - lng1);
    
    const a = 
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(this.toRadians(lat1)) * Math.cos(this.toRadians(lat2)) *
      Math.sin(dLng / 2) * Math.sin(dLng / 2);
    
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    
    return R * c;
  }

  private toRadians(degrees: number): number {
    return degrees * (Math.PI / 180);
  }

  // Event listeners
  onPositionUpdate(callback: (position: LocationData) => void): () => void {
    this.positionListeners.add(callback);
    
    return () => {
      this.positionListeners.delete(callback);
    };
  }

  onError(callback: (error: GeolocationPositionError) => void): () => void {
    this.errorListeners.add(callback);
    
    return () => {
      this.errorListeners.delete(callback);
    };
  }

  private notifyPositionListeners(position: LocationData): void {
    this.positionListeners.forEach(callback => {
      try {
        callback(position);
      } catch (error) {
        console.error('Position listener error:', error);
      }
    });
  }

  private notifyErrorListeners(error: GeolocationPositionError): void {
    this.errorListeners.forEach(callback => {
      try {
        callback(error);
      } catch (listenerError) {
        console.error('Error listener error:', listenerError);
      }
    });
  }

  // Get last known position
  getLastKnownPosition(): LocationData | null {
    return this.lastKnownPosition;
  }
}

// Device Orientation and Motion Manager
interface OrientationData {
  alpha: number | null; // Z-axis rotation (compass)
  beta: number | null;  // X-axis rotation (front-to-back tilt)
  gamma: number | null; // Y-axis rotation (left-to-right tilt)
  absolute: boolean;
}

interface MotionData {
  acceleration: {
    x: number | null;
    y: number | null;
    z: number | null;
  };
  accelerationIncludingGravity: {
    x: number | null;
    y: number | null;
    z: number | null;
  };
  rotationRate: {
    alpha: number | null;
    beta: number | null;
    gamma: number | null;
  };
  interval: number;
}

class DeviceSensorManager {
  private orientationListeners = new Set<(data: OrientationData) => void>();
  private motionListeners = new Set<(data: MotionData) => void>();
  private isListening = false;

  // Start listening to device sensors
  async startListening(): Promise<void> {
    if (this.isListening) {
      return;
    }

    // Request permission for iOS devices
    if (typeof DeviceOrientationEvent.requestPermission === 'function') {
      const permission = await DeviceOrientationEvent.requestPermission();
      if (permission !== 'granted') {
        throw new Error('Device orientation permission denied');
      }
    }

    if (typeof DeviceMotionEvent.requestPermission === 'function') {
      const permission = await DeviceMotionEvent.requestPermission();
      if (permission !== 'granted') {
        throw new Error('Device motion permission denied');
      }
    }

    // Add event listeners
    window.addEventListener('deviceorientation', this.handleOrientation);
    window.addEventListener('devicemotion', this.handleMotion);
    
    this.isListening = true;
  }

  // Stop listening to device sensors
  stopListening(): void {
    if (!this.isListening) {
      return;
    }

    window.removeEventListener('deviceorientation', this.handleOrientation);
    window.removeEventListener('devicemotion', this.handleMotion);
    
    this.isListening = false;
  }

  private handleOrientation = (event: DeviceOrientationEvent): void => {
    const orientationData: OrientationData = {
      alpha: event.alpha,
      beta: event.beta,
      gamma: event.gamma,
      absolute: event.absolute,
    };

    this.notifyOrientationListeners(orientationData);
  };

  private handleMotion = (event: DeviceMotionEvent): void => {
    const motionData: MotionData = {
      acceleration: {
        x: event.acceleration?.x || null,
        y: event.acceleration?.y || null,
        z: event.acceleration?.z || null,
      },
      accelerationIncludingGravity: {
        x: event.accelerationIncludingGravity?.x || null,
        y: event.accelerationIncludingGravity?.y || null,
        z: event.accelerationIncludingGravity?.z || null,
      },
      rotationRate: {
        alpha: event.rotationRate?.alpha || null,
        beta: event.rotationRate?.beta || null,
        gamma: event.rotationRate?.gamma || null,
      },
      interval: event.interval,
    };

    this.notifyMotionListeners(motionData);
  };

  // Event listeners
  onOrientationChange(callback: (data: OrientationData) => void): () => void {
    this.orientationListeners.add(callback);
    
    return () => {
      this.orientationListeners.delete(callback);
    };
  }

  onMotionChange(callback: (data: MotionData) => void): () => void {
    this.motionListeners.add(callback);
    
    return () => {
      this.motionListeners.delete(callback);
    };
  }

  private notifyOrientationListeners(data: OrientationData): void {
    this.orientationListeners.forEach(callback => {
      try {
        callback(data);
      } catch (error) {
        console.error('Orientation listener error:', error);
      }
    });
  }

  private notifyMotionListeners(data: MotionData): void {
    this.motionListeners.forEach(callback => {
      try {
        callback(data);
      } catch (error) {
        console.error('Motion listener error:', error);
      }
    });
  }

  // Check if sensors are supported
  static isOrientationSupported(): boolean {
    return 'DeviceOrientationEvent' in window;
  }

  static isMotionSupported(): boolean {
    return 'DeviceMotionEvent' in window;
  }

  // Utility functions
  getCompassHeading(alpha: number | null): string {
    if (alpha === null) return 'Unknown';

    const directions = ['N', 'NNE', 'NE', 'ENE', 'E', 'ESE', 'SE', 'SSE', 'S', 'SSW', 'SW', 'WSW', 'W', 'WNW', 'NW', 'NNW'];
    const index = Math.round(alpha / 22.5) % 16;
    return directions[index];
  }

  getTiltDirection(beta: number | null, gamma: number | null): {
    frontBack: 'forward' | 'backward' | 'level';
    leftRight: 'left' | 'right' | 'level';
  } {
    const frontBack = beta === null ? 'level' :
                      beta > 10 ? 'forward' :
                      beta < -10 ? 'backward' : 'level';

    const leftRight = gamma === null ? 'level' :
                      gamma > 10 ? 'left' :
                      gamma < -10 ? 'right' : 'level';

    return { frontBack, leftRight };
  }
}

// React hooks for geolocation and sensors
export function useGeolocation(config: GeolocationConfig = {}) {
  const [geoManager] = useState(() => new GeolocationManager(config));
  const [position, setPosition] = useState<LocationData | null>(null);
  const [error, setError] = useState<GeolocationPositionError | null>(null);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    const unsubscribePosition = geoManager.onPositionUpdate(setPosition);
    const unsubscribeError = geoManager.onError(setError);

    if (config.watchPosition) {
      geoManager.startWatching();
    }

    return () => {
      unsubscribePosition();
      unsubscribeError();
      geoManager.stopWatching();
    };
  }, [geoManager, config.watchPosition]);

  const getCurrentPosition = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    
    try {
      const pos = await geoManager.getCurrentPosition();
      setPosition(pos);
      return pos;
    } catch (err) {
      setError(err as GeolocationPositionError);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, [geoManager]);

  const calculateDistance = useCallback((lat: number, lng: number, unit?: 'km' | 'miles') => {
    if (!position) return null;
    return geoManager.calculateDistance(position.latitude, position.longitude, lat, lng, unit);
  }, [geoManager, position]);

  return {
    position,
    error,
    isLoading,
    getCurrentPosition,
    calculateDistance,
    lastKnownPosition: geoManager.getLastKnownPosition(),
  };
}

export function useDeviceSensors() {
  const [sensorManager] = useState(() => new DeviceSensorManager());
  const [orientation, setOrientation] = useState<OrientationData | null>(null);
  const [motion, setMotion] = useState<MotionData | null>(null);
  const [isListening, setIsListening] = useState(false);

  useEffect(() => {
    const unsubscribeOrientation = sensorManager.onOrientationChange(setOrientation);
    const unsubscribeMotion = sensorManager.onMotionChange(setMotion);

    return () => {
      unsubscribeOrientation();
      unsubscribeMotion();
      sensorManager.stopListening();
    };
  }, [sensorManager]);

  const startListening = useCallback(async () => {
    try {
      await sensorManager.startListening();
      setIsListening(true);
    } catch (error) {
      console.error('Sensor listening failed:', error);
      throw error;
    }
  }, [sensorManager]);

  const stopListening = useCallback(() => {
    sensorManager.stopListening();
    setIsListening(false);
  }, [sensorManager]);

  return {
    orientation,
    motion,
    isListening,
    startListening,
    stopListening,
    isOrientationSupported: DeviceSensorManager.isOrientationSupported(),
    isMotionSupported: DeviceSensorManager.isMotionSupported(),
    getCompassHeading: (alpha: number | null) => sensorManager.getCompassHeading(alpha),
    getTiltDirection: (beta: number | null, gamma: number | null) => 
      sensorManager.getTiltDirection(beta, gamma),
  };
}
```

## Interview-Ready Summary

**Modern Web APIs provide advanced capabilities:**

1. **Storage APIs** - IndexedDB for complex data, enhanced localStorage/sessionStorage with expiry, storage quota management, persistent storage
2. **Media APIs** - Camera/microphone access, screen capture, media recording, image processing, device enumeration
3. **Geolocation APIs** - Position tracking, address resolution, distance calculation, fallback providers
4. **Device Sensors** - Device orientation, motion detection, compass heading, tilt detection

**Key implementation patterns:**
- **Storage Management** - Multi-tier storage strategy, automatic cleanup, quota monitoring, version management
- **Media Capture** - Progressive enhancement, device switching, filter application, error handling
- **Location Services** - High accuracy positioning, background tracking, reverse geocoding, privacy handling
- **Sensor Integration** - Permission management, cross-platform compatibility, sensor fusion

**Browser API capabilities:** Offline data storage, real-time media processing, location-aware applications, motion-responsive interfaces.

**Best practices:** Handle permissions gracefully, provide fallbacks, implement progressive enhancement, manage resource cleanup, optimize for mobile devices, respect user privacy, handle API limitations, monitor performance impact.