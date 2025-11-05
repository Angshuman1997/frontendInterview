# Data Privacy and GDPR Compliance for Frontend Applications

The General Data Protection Regulation (GDPR) and other privacy laws require frontend applications to implement comprehensive data protection measures. This guide covers GDPR compliance strategies, data privacy patterns, and privacy-by-design implementations for modern web applications.

## GDPR Fundamentals for Frontend Developers

### Understanding GDPR Requirements

```typescript
// GDPR Data Categories and Processing
interface PersonalData {
  category: 'basic' | 'sensitive' | 'biometric' | 'health' | 'financial';
  fields: string[];
  retention: number; // days
  processingLawfulBasis: LawfulBasis;
  consentRequired: boolean;
  anonymizable: boolean;
}

type LawfulBasis = 
  | 'consent'
  | 'contract'
  | 'legal_obligation'
  | 'vital_interests'
  | 'public_task'
  | 'legitimate_interests';

interface DataProcessingRecord {
  id: string;
  dataCategory: string;
  processingPurpose: string;
  lawfulBasis: LawfulBasis;
  dataSubject: string;
  timestamp: string;
  retention: Date;
  thirdPartySharing: boolean;
  crossBorderTransfer: boolean;
}

// GDPR Compliance Manager
class GDPRComplianceManager {
  private processingRecords: Map<string, DataProcessingRecord> = new Map();
  private dataCategories: Map<string, PersonalData> = new Map();
  private consentManager: ConsentManager;

  constructor() {
    this.consentManager = new ConsentManager();
    this.initializeDataCategories();
  }

  // Register data processing activity
  registerProcessing(
    dataCategory: string,
    purpose: string,
    lawfulBasis: LawfulBasis,
    dataSubject: string,
    options: {
      thirdPartySharing?: boolean;
      crossBorderTransfer?: boolean;
      customRetention?: number;
    } = {}
  ): string {
    const category = this.dataCategories.get(dataCategory);
    if (!category) {
      throw new Error(`Unknown data category: ${dataCategory}`);
    }

    // Check if consent is required
    if (category.consentRequired && lawfulBasis === 'consent') {
      const hasConsent = this.consentManager.hasValidConsent(dataSubject, purpose);
      if (!hasConsent) {
        throw new Error('Valid consent required for this processing activity');
      }
    }

    const recordId = this.generateRecordId();
    const retentionDays = options.customRetention || category.retention;
    
    const record: DataProcessingRecord = {
      id: recordId,
      dataCategory,
      processingPurpose: purpose,
      lawfulBasis,
      dataSubject,
      timestamp: new Date().toISOString(),
      retention: new Date(Date.now() + retentionDays * 24 * 60 * 60 * 1000),
      thirdPartySharing: options.thirdPartySharing || false,
      crossBorderTransfer: options.crossBorderTransfer || false
    };

    this.processingRecords.set(recordId, record);
    return recordId;
  }

  // Check data retention compliance
  checkRetentionCompliance(): DataProcessingRecord[] {
    const expiredRecords: DataProcessingRecord[] = [];
    const now = new Date();

    this.processingRecords.forEach(record => {
      if (record.retention < now) {
        expiredRecords.push(record);
      }
    });

    return expiredRecords;
  }

  // Data subject rights implementation
  async exerciseDataSubjectRights(
    dataSubject: string,
    right: 'access' | 'rectification' | 'erasure' | 'portability' | 'restriction' | 'objection'
  ): Promise<any> {
    switch (right) {
      case 'access':
        return this.provideDataAccess(dataSubject);
      case 'rectification':
        return this.enableDataRectification(dataSubject);
      case 'erasure':
        return this.processDataErasure(dataSubject);
      case 'portability':
        return this.provideDataPortability(dataSubject);
      case 'restriction':
        return this.restrictProcessing(dataSubject);
      case 'objection':
        return this.processObjection(dataSubject);
      default:
        throw new Error(`Unsupported data subject right: ${right}`);
    }
  }

  // Data anonymization
  anonymizeData(data: any, category: string): any {
    const categoryInfo = this.dataCategories.get(category);
    if (!categoryInfo || !categoryInfo.anonymizable) {
      throw new Error('Data category cannot be anonymized');
    }

    return this.applyAnonymizationTechniques(data, categoryInfo.fields);
  }

  private initializeDataCategories(): void {
    // Basic personal data
    this.dataCategories.set('basic_personal', {
      category: 'basic',
      fields: ['name', 'email', 'phone', 'address'],
      retention: 365 * 2, // 2 years
      processingLawfulBasis: 'consent',
      consentRequired: true,
      anonymizable: true
    });

    // Sensitive personal data
    this.dataCategories.set('sensitive_personal', {
      category: 'sensitive',
      fields: ['race', 'religion', 'political_opinions', 'health_data'],
      retention: 365, // 1 year
      processingLawfulBasis: 'consent',
      consentRequired: true,
      anonymizable: false
    });

    // Technical data
    this.dataCategories.set('technical', {
      category: 'basic',
      fields: ['ip_address', 'cookies', 'device_id', 'user_agent'],
      retention: 365, // 1 year
      processingLawfulBasis: 'legitimate_interests',
      consentRequired: false,
      anonymizable: true
    });
  }

  private async provideDataAccess(dataSubject: string): Promise<any> {
    const records = Array.from(this.processingRecords.values())
      .filter(record => record.dataSubject === dataSubject);

    return {
      personalData: await this.gatherPersonalData(dataSubject),
      processingActivities: records,
      thirdPartySharing: records.filter(r => r.thirdPartySharing),
      retentionPeriods: records.map(r => ({
        category: r.dataCategory,
        retention: r.retention
      }))
    };
  }

  private async enableDataRectification(dataSubject: string): Promise<any> {
    // Implementation would provide interface for data correction
    return {
      rectificationForm: '/api/gdpr/rectification',
      currentData: await this.gatherPersonalData(dataSubject)
    };
  }

  private async processDataErasure(dataSubject: string): Promise<any> {
    // Implementation would handle right to be forgotten
    const recordsToDelete = Array.from(this.processingRecords.values())
      .filter(record => record.dataSubject === dataSubject);

    // Check if erasure is possible (e.g., legal obligations might prevent it)
    const canErase = recordsToDelete.every(record => 
      record.lawfulBasis !== 'legal_obligation'
    );

    if (!canErase) {
      throw new Error('Data cannot be erased due to legal obligations');
    }

    // Delete records
    recordsToDelete.forEach(record => {
      this.processingRecords.delete(record.id);
    });

    return { deleted: recordsToDelete.length };
  }

  private async provideDataPortability(dataSubject: string): Promise<any> {
    const personalData = await this.gatherPersonalData(dataSubject);
    
    return {
      format: 'JSON',
      data: personalData,
      downloadUrl: `/api/gdpr/export/${dataSubject}`
    };
  }

  private async restrictProcessing(dataSubject: string): Promise<any> {
    // Mark records as restricted
    this.processingRecords.forEach(record => {
      if (record.dataSubject === dataSubject) {
        (record as any).restricted = true;
      }
    });

    return { restricted: true };
  }

  private async processObjection(dataSubject: string): Promise<any> {
    // Handle objection to processing
    const affectedRecords = Array.from(this.processingRecords.values())
      .filter(record => 
        record.dataSubject === dataSubject && 
        record.lawfulBasis === 'legitimate_interests'
      );

    // Stop processing based on legitimate interests
    affectedRecords.forEach(record => {
      this.processingRecords.delete(record.id);
    });

    return { stopped: affectedRecords.length };
  }

  private async gatherPersonalData(dataSubject: string): Promise<any> {
    // Implementation would collect all personal data for the subject
    return {
      basic: { /* user's basic data */ },
      preferences: { /* user's preferences */ },
      activity: { /* user's activity data */ }
    };
  }

  private applyAnonymizationTechniques(data: any, fields: string[]): any {
    const anonymized = { ...data };
    
    fields.forEach(field => {
      if (anonymized[field]) {
        // Apply different anonymization techniques based on field type
        switch (field) {
          case 'email':
            anonymized[field] = this.anonymizeEmail(anonymized[field]);
            break;
          case 'name':
            anonymized[field] = this.anonymizeName(anonymized[field]);
            break;
          case 'ip_address':
            anonymized[field] = this.anonymizeIP(anonymized[field]);
            break;
          default:
            anonymized[field] = '[ANONYMIZED]';
        }
      }
    });

    return anonymized;
  }

  private anonymizeEmail(email: string): string {
    const [localPart, domain] = email.split('@');
    const anonymizedLocal = localPart.charAt(0) + '*'.repeat(localPart.length - 2) + localPart.charAt(localPart.length - 1);
    return `${anonymizedLocal}@${domain}`;
  }

  private anonymizeName(name: string): string {
    const parts = name.split(' ');
    return parts.map(part => 
      part.charAt(0) + '*'.repeat(part.length - 1)
    ).join(' ');
  }

  private anonymizeIP(ip: string): string {
    const parts = ip.split('.');
    if (parts.length === 4) {
      return `${parts[0]}.${parts[1]}.XXX.XXX`;
    }
    return 'XXX.XXX.XXX.XXX';
  }

  private generateRecordId(): string {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

## Consent Management Implementation

### Advanced Consent Management System

```typescript
// Consent Management
interface ConsentRecord {
  id: string;
  dataSubject: string;
  purposes: string[];
  lawfulBasis: LawfulBasis;
  givenAt: string;
  expiresAt?: string;
  withdrawnAt?: string;
  version: string;
  ipAddress: string;
  userAgent: string;
  granular: boolean;
  preferences: ConsentPreferences;
}

interface ConsentPreferences {
  marketing: boolean;
  analytics: boolean;
  functional: boolean;
  necessary: boolean;
  thirdParty: boolean;
  crossBorder: boolean;
}

interface ConsentBanner {
  purposes: ConsentPurpose[];
  legalBasis: string;
  retentionPeriod: string;
  thirdParties: string[];
  privacyPolicyUrl: string;
  allowGranular: boolean;
}

interface ConsentPurpose {
  id: string;
  name: string;
  description: string;
  required: boolean;
  category: 'necessary' | 'functional' | 'analytics' | 'marketing' | 'personalization';
}

class ConsentManager {
  private consents: Map<string, ConsentRecord> = new Map();
  private consentBanner: ConsentBanner;

  constructor(bannerConfig: ConsentBanner) {
    this.consentBanner = bannerConfig;
  }

  // Record consent
  recordConsent(
    dataSubject: string,
    purposes: string[],
    preferences: ConsentPreferences,
    metadata: {
      ipAddress: string;
      userAgent: string;
      version: string;
    }
  ): string {
    const consentId = this.generateConsentId();
    const now = new Date();
    const expiresAt = new Date(now.getTime() + 365 * 24 * 60 * 60 * 1000); // 1 year

    const consent: ConsentRecord = {
      id: consentId,
      dataSubject,
      purposes,
      lawfulBasis: 'consent',
      givenAt: now.toISOString(),
      expiresAt: expiresAt.toISOString(),
      version: metadata.version,
      ipAddress: metadata.ipAddress,
      userAgent: metadata.userAgent,
      granular: true,
      preferences
    };

    this.consents.set(consentId, consent);
    
    // Store in persistent storage
    this.persistConsent(consent);
    
    return consentId;
  }

  // Check if valid consent exists
  hasValidConsent(dataSubject: string, purpose: string): boolean {
    const userConsents = Array.from(this.consents.values())
      .filter(consent => 
        consent.dataSubject === dataSubject &&
        !consent.withdrawnAt &&
        this.isConsentValid(consent)
      );

    return userConsents.some(consent => 
      consent.purposes.includes(purpose)
    );
  }

  // Withdraw consent
  withdrawConsent(dataSubject: string, purposes?: string[]): void {
    const userConsents = Array.from(this.consents.values())
      .filter(consent => consent.dataSubject === dataSubject);

    userConsents.forEach(consent => {
      if (!purposes || purposes.some(p => consent.purposes.includes(p))) {
        consent.withdrawnAt = new Date().toISOString();
        this.persistConsent(consent);
      }
    });
  }

  // Update consent preferences
  updateConsent(
    dataSubject: string,
    newPreferences: Partial<ConsentPreferences>
  ): string {
    // Withdraw existing consents
    this.withdrawConsent(dataSubject);

    // Create new consent with updated preferences
    const currentConsent = this.getCurrentConsent(dataSubject);
    const updatedPreferences = {
      ...currentConsent?.preferences,
      ...newPreferences
    };

    const purposes = this.getActivePurposes(updatedPreferences);
    
    return this.recordConsent(
      dataSubject,
      purposes,
      updatedPreferences as ConsentPreferences,
      {
        ipAddress: 'updated',
        userAgent: 'updated',
        version: '2.0'
      }
    );
  }

  // Get consent history
  getConsentHistory(dataSubject: string): ConsentRecord[] {
    return Array.from(this.consents.values())
      .filter(consent => consent.dataSubject === dataSubject)
      .sort((a, b) => new Date(b.givenAt).getTime() - new Date(a.givenAt).getTime());
  }

  // Generate consent report
  generateConsentReport(dataSubject: string): any {
    const history = this.getConsentHistory(dataSubject);
    const current = this.getCurrentConsent(dataSubject);

    return {
      currentConsent: current,
      consentHistory: history,
      purposes: current?.purposes || [],
      preferences: current?.preferences || {},
      withdrawalRights: {
        canWithdraw: true,
        methods: ['online', 'email', 'postal']
      },
      dataRetention: {
        period: '1 year after withdrawal',
        automaticDeletion: true
      }
    };
  }

  private isConsentValid(consent: ConsentRecord): boolean {
    if (consent.withdrawnAt) return false;
    
    if (consent.expiresAt) {
      const expires = new Date(consent.expiresAt);
      return expires > new Date();
    }
    
    return true;
  }

  private getCurrentConsent(dataSubject: string): ConsentRecord | null {
    const userConsents = Array.from(this.consents.values())
      .filter(consent => 
        consent.dataSubject === dataSubject &&
        this.isConsentValid(consent)
      );

    return userConsents.sort((a, b) => 
      new Date(b.givenAt).getTime() - new Date(a.givenAt).getTime()
    )[0] || null;
  }

  private getActivePurposes(preferences: ConsentPreferences): string[] {
    const purposes: string[] = [];
    
    if (preferences.necessary) purposes.push('necessary');
    if (preferences.functional) purposes.push('functional');
    if (preferences.analytics) purposes.push('analytics');
    if (preferences.marketing) purposes.push('marketing');
    if (preferences.thirdParty) purposes.push('third_party');
    
    return purposes;
  }

  private persistConsent(consent: ConsentRecord): void {
    // In production, this would save to secure backend storage
    localStorage.setItem(`consent_${consent.id}`, JSON.stringify(consent));
  }

  private generateConsentId(): string {
    return `consent_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// React Consent Banner Component
export const ConsentBanner: React.FC<{
  onConsentGiven: (preferences: ConsentPreferences) => void;
  onConsentDeclined: () => void;
  allowGranular?: boolean;
}> = ({ onConsentGiven, onConsentDeclined, allowGranular = true }) => {
  const [showBanner, setShowBanner] = useState(false);
  const [showDetails, setShowDetails] = useState(false);
  const [preferences, setPreferences] = useState<ConsentPreferences>({
    marketing: false,
    analytics: false,
    functional: true,
    necessary: true,
    thirdParty: false,
    crossBorder: false
  });

  useEffect(() => {
    // Check if consent already exists
    const hasConsent = localStorage.getItem('user_consent');
    setShowBanner(!hasConsent);
  }, []);

  const handleAcceptAll = () => {
    const allAccepted: ConsentPreferences = {
      marketing: true,
      analytics: true,
      functional: true,
      necessary: true,
      thirdParty: true,
      crossBorder: true
    };
    
    onConsentGiven(allAccepted);
    setShowBanner(false);
    localStorage.setItem('user_consent', JSON.stringify(allAccepted));
  };

  const handleAcceptSelected = () => {
    onConsentGiven(preferences);
    setShowBanner(false);
    localStorage.setItem('user_consent', JSON.stringify(preferences));
  };

  const handleDeclineAll = () => {
    const minimal: ConsentPreferences = {
      marketing: false,
      analytics: false,
      functional: false,
      necessary: true,
      thirdParty: false,
      crossBorder: false
    };
    
    onConsentDeclined();
    setShowBanner(false);
    localStorage.setItem('user_consent', JSON.stringify(minimal));
  };

  if (!showBanner) return null;

  return (
    <div className="consent-banner">
      <div className="consent-content">
        <h3>Cookie and Privacy Preferences</h3>
        <p>
          We use cookies and similar technologies to provide, protect, and improve our services. 
          You can choose which categories of cookies to accept.
        </p>
        
        {showDetails && allowGranular && (
          <div className="consent-details">
            <div className="consent-category">
              <label>
                <input
                  type="checkbox"
                  checked={preferences.necessary}
                  onChange={(e) => setPreferences(prev => ({ 
                    ...prev, 
                    necessary: e.target.checked 
                  }))}
                  disabled
                />
                Necessary Cookies (Required)
              </label>
              <p>Essential for the website to function properly.</p>
            </div>
            
            <div className="consent-category">
              <label>
                <input
                  type="checkbox"
                  checked={preferences.functional}
                  onChange={(e) => setPreferences(prev => ({ 
                    ...prev, 
                    functional: e.target.checked 
                  }))}
                />
                Functional Cookies
              </label>
              <p>Enable enhanced functionality and personalization.</p>
            </div>
            
            <div className="consent-category">
              <label>
                <input
                  type="checkbox"
                  checked={preferences.analytics}
                  onChange={(e) => setPreferences(prev => ({ 
                    ...prev, 
                    analytics: e.target.checked 
                  }))}
                />
                Analytics Cookies
              </label>
              <p>Help us understand how visitors use our website.</p>
            </div>
            
            <div className="consent-category">
              <label>
                <input
                  type="checkbox"
                  checked={preferences.marketing}
                  onChange={(e) => setPreferences(prev => ({ 
                    ...prev, 
                    marketing: e.target.checked 
                  }))}
                />
                Marketing Cookies
              </label>
              <p>Used to deliver relevant advertisements.</p>
            </div>
            
            <div className="consent-category">
              <label>
                <input
                  type="checkbox"
                  checked={preferences.thirdParty}
                  onChange={(e) => setPreferences(prev => ({ 
                    ...prev, 
                    thirdParty: e.target.checked 
                  }))}
                />
                Third-Party Cookies
              </label>
              <p>Allow third-party services to set cookies.</p>
            </div>
          </div>
        )}
        
        <div className="consent-actions">
          <button onClick={handleAcceptAll}>Accept All</button>
          
          {allowGranular && (
            <>
              <button onClick={() => setShowDetails(!showDetails)}>
                {showDetails ? 'Hide' : 'Customize'}
              </button>
              
              {showDetails && (
                <button onClick={handleAcceptSelected}>
                  Accept Selected
                </button>
              )}
            </>
          )}
          
          <button onClick={handleDeclineAll}>Decline All</button>
        </div>
        
        <div className="consent-links">
          <a href="/privacy-policy">Privacy Policy</a>
          <a href="/cookie-policy">Cookie Policy</a>
          <a href="/gdpr-rights">Your Rights</a>
        </div>
      </div>
    </div>
  );
};

// GDPR Rights Management Component
export const GDPRRightsManager: React.FC<{
  dataSubject: string;
}> = ({ dataSubject }) => {
  const [activeRequest, setActiveRequest] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [gdprManager] = useState(() => new GDPRComplianceManager());

  const handleDataSubjectRequest = async (right: string) => {
    setIsLoading(true);
    setActiveRequest(right);
    
    try {
      const result = await gdprManager.exerciseDataSubjectRights(
        dataSubject,
        right as any
      );
      
      console.log(`${right} request completed:`, result);
      
      // Show success notification
      alert(`${right} request has been processed successfully.`);
      
    } catch (error) {
      console.error(`${right} request failed:`, error);
      alert(`${right} request failed. Please try again or contact support.`);
    } finally {
      setIsLoading(false);
      setActiveRequest(null);
    }
  };

  return (
    <div className="gdpr-rights-manager">
      <h3>Your Data Protection Rights</h3>
      <p>Under GDPR, you have the following rights regarding your personal data:</p>
      
      <div className="rights-list">
        <div className="right-item">
          <h4>Right of Access</h4>
          <p>Request a copy of your personal data we hold.</p>
          <button
            onClick={() => handleDataSubjectRequest('access')}
            disabled={isLoading && activeRequest === 'access'}
          >
            {isLoading && activeRequest === 'access' ? 'Processing...' : 'Request Data Access'}
          </button>
        </div>
        
        <div className="right-item">
          <h4>Right to Rectification</h4>
          <p>Correct inaccurate or incomplete personal data.</p>
          <button
            onClick={() => handleDataSubjectRequest('rectification')}
            disabled={isLoading && activeRequest === 'rectification'}
          >
            {isLoading && activeRequest === 'rectification' ? 'Processing...' : 'Request Data Correction'}
          </button>
        </div>
        
        <div className="right-item">
          <h4>Right to Erasure</h4>
          <p>Request deletion of your personal data (right to be forgotten).</p>
          <button
            onClick={() => handleDataSubjectRequest('erasure')}
            disabled={isLoading && activeRequest === 'erasure'}
          >
            {isLoading && activeRequest === 'erasure' ? 'Processing...' : 'Request Data Deletion'}
          </button>
        </div>
        
        <div className="right-item">
          <h4>Right to Data Portability</h4>
          <p>Receive your personal data in a structured, machine-readable format.</p>
          <button
            onClick={() => handleDataSubjectRequest('portability')}
            disabled={isLoading && activeRequest === 'portability'}
          >
            {isLoading && activeRequest === 'portability' ? 'Processing...' : 'Download My Data'}
          </button>
        </div>
        
        <div className="right-item">
          <h4>Right to Restrict Processing</h4>
          <p>Limit how we process your personal data.</p>
          <button
            onClick={() => handleDataSubjectRequest('restriction')}
            disabled={isLoading && activeRequest === 'restriction'}
          >
            {isLoading && activeRequest === 'restriction' ? 'Processing...' : 'Restrict Processing'}
          </button>
        </div>
        
        <div className="right-item">
          <h4>Right to Object</h4>
          <p>Object to processing based on legitimate interests.</p>
          <button
            onClick={() => handleDataSubjectRequest('objection')}
            disabled={isLoading && activeRequest === 'objection'}
          >
            {isLoading && activeRequest === 'objection' ? 'Processing...' : 'Object to Processing'}
          </button>
        </div>
      </div>
    </div>
  );
};
```

## Privacy-by-Design Implementation

### Privacy-First Component Architecture

```typescript
// Privacy-by-Design Patterns
interface PrivacyConfig {
  dataMinimization: boolean;
  purposeLimitation: boolean;
  storageMinimization: boolean;
  transparencyRequired: boolean;
  consentRequired: boolean;
  anonymizationLevel: 'none' | 'pseudonymous' | 'anonymous';
}

// Privacy-aware data collection hook
export const usePrivacyAwareData = <T>(
  initialData: T,
  privacyConfig: PrivacyConfig
) => {
  const [data, setData] = useState<T>(initialData);
  const [privacyStatus, setPrivacyStatus] = useState({
    consentGiven: false,
    dataMinimized: false,
    anonymized: false
  });

  const updateData = useCallback((newData: Partial<T>, purpose: string) => {
    // Check consent for the specific purpose
    if (privacyConfig.consentRequired) {
      const hasConsent = checkPurposeConsent(purpose);
      if (!hasConsent) {
        throw new Error(`Consent required for purpose: ${purpose}`);
      }
    }

    // Apply data minimization
    let processedData = newData;
    if (privacyConfig.dataMinimization) {
      processedData = minimizeDataForPurpose(newData, purpose);
    }

    // Apply anonymization
    if (privacyConfig.anonymizationLevel !== 'none') {
      processedData = anonymizeData(processedData, privacyConfig.anonymizationLevel);
    }

    setData(prevData => ({ ...prevData, ...processedData }));
    
    // Update privacy status
    setPrivacyStatus({
      consentGiven: true,
      dataMinimized: privacyConfig.dataMinimization,
      anonymized: privacyConfig.anonymizationLevel !== 'none'
    });
  }, [privacyConfig]);

  return {
    data,
    updateData,
    privacyStatus,
    isPrivacyCompliant: privacyStatus.consentGiven
  };
};

// Privacy impact assessment
const conductPrivacyImpactAssessment = (
  component: string,
  dataTypes: string[],
  purposes: string[]
): PrivacyImpactAssessment => {
  const riskLevel = calculatePrivacyRisk(dataTypes, purposes);
  const mitigationMeasures = recommendMitigations(riskLevel, dataTypes);
  
  return {
    component,
    riskLevel,
    dataTypes,
    purposes,
    mitigationMeasures,
    complianceStatus: riskLevel <= 'medium' ? 'compliant' : 'requires_review',
    reviewDate: new Date(Date.now() + 90 * 24 * 60 * 60 * 1000) // 90 days
  };
};

interface PrivacyImpactAssessment {
  component: string;
  riskLevel: 'low' | 'medium' | 'high' | 'very_high';
  dataTypes: string[];
  purposes: string[];
  mitigationMeasures: string[];
  complianceStatus: 'compliant' | 'requires_review' | 'non_compliant';
  reviewDate: Date;
}

// Helper functions
const checkPurposeConsent = (purpose: string): boolean => {
  const consent = JSON.parse(localStorage.getItem('user_consent') || '{}');
  return consent[purpose] === true;
};

const minimizeDataForPurpose = <T>(data: Partial<T>, purpose: string): Partial<T> => {
  // Define which fields are necessary for each purpose
  const purposeFields: Record<string, string[]> = {
    'analytics': ['userId', 'sessionId', 'timestamp'],
    'marketing': ['email', 'preferences'],
    'functional': ['userId', 'settings', 'preferences'],
    'necessary': ['sessionId', 'csrf_token']
  };

  const allowedFields = purposeFields[purpose] || [];
  const minimized: Partial<T> = {};

  Object.keys(data).forEach(key => {
    if (allowedFields.includes(key)) {
      minimized[key as keyof T] = data[key as keyof T];
    }
  });

  return minimized;
};

const anonymizeData = <T>(data: Partial<T>, level: 'pseudonymous' | 'anonymous'): Partial<T> => {
  const anonymized = { ...data };

  Object.keys(anonymized).forEach(key => {
    const value = anonymized[key as keyof T];
    
    if (typeof value === 'string') {
      if (level === 'pseudonymous') {
        // Replace with pseudonym
        anonymized[key as keyof T] = `pseudo_${hashString(value)}` as T[keyof T];
      } else if (level === 'anonymous') {
        // Remove or generalize
        anonymized[key as keyof T] = '[ANONYMIZED]' as T[keyof T];
      }
    }
  });

  return anonymized;
};

const hashString = (str: string): string => {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i);
    hash = ((hash << 5) - hash) + char;
    hash = hash & hash; // Convert to 32-bit integer
  }
  return Math.abs(hash).toString(36);
};

const calculatePrivacyRisk = (dataTypes: string[], purposes: string[]): 'low' | 'medium' | 'high' | 'very_high' => {
  const sensitiveDataTypes = ['email', 'phone', 'address', 'biometric', 'health', 'financial'];
  const highRiskPurposes = ['profiling', 'automated_decision_making', 'cross_border_transfer'];

  const hasSensitiveData = dataTypes.some(type => sensitiveDataTypes.includes(type));
  const hasHighRiskPurpose = purposes.some(purpose => highRiskPurposes.includes(purpose));

  if (hasSensitiveData && hasHighRiskPurpose) return 'very_high';
  if (hasSensitiveData || hasHighRiskPurpose) return 'high';
  if (dataTypes.length > 5) return 'medium';
  return 'low';
};

const recommendMitigations = (riskLevel: string, dataTypes: string[]): string[] => {
  const baseMitigations = [
    'Implement data minimization',
    'Ensure lawful basis for processing',
    'Provide clear privacy notice'
  ];

  const additionalMitigations: Record<string, string[]> = {
    'medium': ['Implement pseudonymization', 'Regular privacy reviews'],
    'high': ['Conduct DPIA', 'Implement encryption', 'Access controls'],
    'very_high': ['Consult with DPO', 'Consider Privacy Enhancing Technologies', 'Regular audits']
  };

  return [...baseMitigations, ...(additionalMitigations[riskLevel] || [])];
};
```

## Interview-Ready GDPR Summary

**Core GDPR Principles:**
1. **Lawfulness, Fairness, Transparency** - Clear legal basis and transparent processing
2. **Purpose Limitation** - Data used only for specified, explicit purposes
3. **Data Minimization** - Collect only necessary data
4. **Accuracy** - Ensure data is accurate and up-to-date
5. **Storage Limitation** - Retain data only as long as necessary
6. **Integrity and Confidentiality** - Secure data processing
7. **Accountability** - Demonstrate compliance

**Frontend Implementation Requirements:**
- Granular consent management with clear opt-in/opt-out
- Data subject rights implementation (access, rectification, erasure, portability)
- Privacy-by-design component architecture
- Data anonymization and pseudonymization techniques
- Audit logging for all data processing activities

**Key Interview Topics:** GDPR compliance strategies, consent management implementation, data subject rights automation, privacy impact assessments, data anonymization techniques, privacy-by-design patterns, cross-border data transfer compliance.