# 生产级人员行为分析Prompt系统

## 系统设计原则
1. **鲁棒性**：处理缺失数据、格式错误、异常输入
2. **自适应**：根据数据质量和用户类型自动调整
3. **可解释**：每个判断都有清晰的依据
4. **高性能**：支持大规模并发处理

---

## Prompt 1: 生产级每日特征提取

### 系统角色
你是一个高可靠的行为特征提取系统，负责从各种质量的日志数据中提取特征。你需要处理不完整数据、识别异常值、标记数据质量问题，并生成标准化的特征报告。

### 输入格式与验证
```json
{
  "user_id": "string, required",
  "date": "YYYY-MM-DD, required",
  "user_context": {
    "is_workday": "boolean, optional, default=true",
    "is_holiday": "boolean, optional, default=false",
    "special_event": "string, optional",  // 如："年终审计"、"系统升级"
    "user_status": "string, optional"     // 如："请假"、"出差"、"在岗"
  },
  "logs": [
    // 日志可能包含各种格式，需要容错处理
    {
      "timestamp": "string or epoch, required",
      "event_type": "string, required",
      // 其他字段都是optional，可能缺失
      "source_ip": "string, optional",
      "target_resource": "string, optional",
      "data_size": "number, optional",
      "result": "string, optional"
    }
  ],
  "data_quality": {
    "log_completeness": "float, 0-1",  // 日志完整度
    "time_coverage": "float, 0-1",     // 时间覆盖率
    "field_missing_rate": "object"     // 各字段缺失率
  }
}
```

### 特征提取规则

#### 1. 数据预处理
```python
处理规则：
1. 时间戳标准化：统一转换为ISO格式
2. IP地址验证：检查格式，标记内外网
3. 缺失值处理：
   - 关键字段缺失：标记为"unknown"
   - 数值字段缺失：使用-1或上下文推断
4. 异常值检测：
   - 超大文件（>1GB）：标记并单独统计
   - 超长会话（>24小时）：标记为异常
5. 去重：相同时间戳的重复事件只计一次
```

#### 2. 特征计算（含容错）
```json
{
  "basic_features": {
    "temporal": {
      "first_activity": "HH:MM:SS or null",
      "last_activity": "HH:MM:SS or null",
      "active_duration_hours": "float or -1",
      "activity_gaps": [  // 活动间隔分析
        {"start": "HH:MM", "end": "HH:MM", "duration_minutes": 120}
      ],
      "coverage_score": 0.8,  // 时间覆盖完整度
      "is_continuous": true,  // 是否连续活动
      "abnormal_hours": []    // 异常时段列表
    },
    
    "volume": {
      "total_events": 150,
      "valid_events": 145,     // 有效事件数
      "error_events": 5,       // 错误事件数
      "events_per_hour": 18.1,
      "peak_hour": "15:00",
      "data_download_mb": 125.5,
      "data_upload_mb": 15.3,
      "data_quality_flag": "partial"  // complete/partial/poor
    },
    
    "access": {
      "unique_resources": 23,
      "resource_categories": {
        "database": 5,
        "file_share": 12,
        "application": 6
      },
      "sensitive_access": {
        "count": 3,
        "resources": ["HR_DB", "FINANCE_SHARE"],
        "risk_level": "medium"
      },
      "cross_system_access": false,
      "new_resource_access": []  // 首次访问的资源
    },
    
    "network": {
      "unique_source_ips": 2,
      "ip_changes": 1,  // IP切换次数
      "internal_traffic_ratio": 0.95,
      "external_destinations": [],
      "vpn_usage": true,
      "suspicious_ports": [],
      "geo_locations": ["北京", "上海"]  // 如果可以解析
    },
    
    "authentication": {
      "login_attempts": 3,
      "failed_logins": 0,
      "auth_methods": ["password", "mfa"],
      "session_count": 2,
      "avg_session_duration_minutes": 240,
      "concurrent_sessions": false
    }
  },
  
  "statistical_features": {
    "entropy": {  // 行为熵值，衡量行为多样性
      "temporal_entropy": 2.3,  // 时间分布熵
      "resource_entropy": 3.1,  // 资源访问熵
      "operation_entropy": 1.8  // 操作类型熵
    },
    "periodicity": {  // 周期性分析
      "daily_pattern_score": 0.85,  // 日模式相似度
      "weekly_pattern_score": 0.72,  // 周模式相似度
      "is_regular": true
    },
    "burstiness": {  // 突发性分析
      "burst_events": [
        {"time": "15:30-15:45", "event_count": 50}
      ],
      "burst_score": 0.3  // 0-1，越高越突发
    }
  },
  
  "contextual_features": {
    "workday_alignment": 0.95,  // 与工作日的符合度
    "peer_similarity": -1,  // 与同组用户的相似度（需要额外输入）
    "role_compliance": -1,  // 与角色的符合度（需要额外输入）
    "temporal_consistency": 0.88  // 时间一致性
  },
  
  "quality_metrics": {
    "data_completeness": 0.85,
    "confidence_score": 0.9,  // 特征可信度
    "missing_fields": ["device_id", "application_name"],
    "anomaly_in_raw_data": ["duplicate_events", "time_gap_2hours"],
    "processing_notes": "20% events missing source_ip, estimated from context"
  }
}
```

### 边界情况处理

#### 情况1：新员工（无历史数据）
```json
{
  "special_handling": "new_user",
  "features": {
    "comparison_baseline": "role_average",  // 使用角色平均值
    "confidence_adjustment": 0.5,  // 降低置信度
    "min_days_required": 7  // 至少需要7天数据才能建立个人基线
  }
}
```

#### 情况2：假期/请假
```json
{
  "special_handling": "absence",
  "features": {
    "expected_activity": false,
    "absence_type": "vacation",
    "should_exclude_from_baseline": true
  }
}
```

#### 情况3：数据严重缺失
```json
{
  "special_handling": "insufficient_data",
  "features": {
    "usable_for_analysis": false,
    "missing_critical_fields": ["timestamp", "event_type"],
    "suggested_action": "request_data_resync"
  }
}
```

---

## 基线计算规则体系

### 统一基线计算框架

```python
class BaselineCalculator:
    """
    生产级基线计算器，支持多种统计方法
    """
    
    def __init__(self, user_profile):
        self.user_profile = user_profile
        self.methods = self._select_methods()
    
    def _select_methods(self):
        """根据用户类型和数据特征选择计算方法"""
        methods = {}
        
        # 时间序列数据：使用移动平均 + 季节性分解
        methods['temporal'] = {
            'algorithm': 'ewma',  # 指数加权移动平均
            'window': 30,
            'seasonality': 'weekly',
            'outlier_method': 'isolation_forest'
        }
        
        # 计数数据：使用中位数和MAD
        methods['count'] = {
            'algorithm': 'median_mad',  # 中位数和绝对中位差
            'robust': True,
            'quantiles': [0.25, 0.75]
        }
        
        # 分类数据：使用频率分布
        methods['categorical'] = {
            'algorithm': 'frequency',
            'min_support': 0.05,  # 最小支持度
            'smoothing': 'laplace'
        }
        
        # 根据岗位调整参数
        if self.user_profile['role'] in ['财务', '人事']:
            methods['sensitivity'] = 'high'  # 更严格的阈值
            methods['temporal']['window'] = 60  # 更长的历史窗口
        elif self.user_profile['role'] in ['销售', '市场']:
            methods['sensitivity'] = 'low'  # 更宽松的阈值
            methods['temporal']['seasonality'] = 'monthly'  # 月度周期
            
        return methods
    
    def calculate_baseline(self, historical_features):
        """
        计算多维度基线
        """
        baseline = {}
        
        # 1. 时间维度基线
        baseline['temporal'] = self._calculate_temporal_baseline(
            [f['temporal'] for f in historical_features]
        )
        
        # 2. 访问维度基线
        baseline['access'] = self._calculate_access_baseline(
            [f['access'] for f in historical_features]
        )
        
        # 3. 流量维度基线
        baseline['volume'] = self._calculate_volume_baseline(
            [f['volume'] for f in historical_features]
        )
        
        # 4. 动态阈值
        baseline['thresholds'] = self._calculate_dynamic_thresholds(
            historical_features
        )
        
        return baseline
    
    def _calculate_temporal_baseline(self, temporal_features):
        """时间模式基线"""
        return {
            'typical_start': {
                'value': self._get_typical_time(
                    [f['first_activity'] for f in temporal_features]
                ),
                'std_minutes': 30,  # 标准差30分钟
                'hard_bounds': ['07:00', '10:00']  # 硬边界
            },
            'typical_end': {
                'value': self._get_typical_time(
                    [f['last_activity'] for f in temporal_features]
                ),
                'std_minutes': 60,
                'hard_bounds': ['17:00', '20:00']
            },
            'active_hours': {
                'mean': np.mean([f['active_duration_hours'] for f in temporal_features]),
                'std': np.std([f['active_duration_hours'] for f in temporal_features]),
                'quantiles': {
                    'q25': np.percentile(..., 25),
                    'q75': np.percentile(..., 75)
                }
            },
            'work_pattern': self._detect_work_pattern(temporal_features)
        }
    
    def _calculate_dynamic_thresholds(self, features):
        """
        动态阈值计算 - 核心创新点
        使用自适应阈值，而不是固定的3-sigma
        """
        thresholds = {}
        
        for metric in ['download_mb', 'upload_mb', 'login_attempts']:
            values = [f.get(metric, 0) for f in features]
            
            # 移除异常值后计算
            cleaned_values = self._remove_outliers(values)
            
            # 计算自适应阈值
            thresholds[metric] = {
                'method': 'adaptive_quantile',
                'normal_range': [
                    np.percentile(cleaned_values, 5),
                    np.percentile(cleaned_values, 95)
                ],
                'warning_threshold': np.percentile(cleaned_values, 98),
                'critical_threshold': np.percentile(cleaned_values, 99.5),
                'adaptive_factor': self._calculate_adaptive_factor(values),
                'confidence': len(cleaned_values) / len(values)  # 数据可信度
            }
            
        return thresholds
    
    def _calculate_adaptive_factor(self, values):
        """
        根据数据分布特征调整阈值敏感度
        """
        cv = np.std(values) / np.mean(values) if np.mean(values) > 0 else 1
        
        if cv < 0.5:  # 低变异性，严格阈值
            return 0.8
        elif cv < 1.0:  # 中等变异性
            return 1.0
        else:  # 高变异性，宽松阈值
            return 1.5
```

### 基线更新策略

```python
class BaselineUpdateStrategy:
    """
    基线更新策略 - 支持多种更新模式
    """
    
    def __init__(self):
        self.update_modes = {
            'incremental': self._incremental_update,  # 增量更新
            'sliding_window': self._sliding_window_update,  # 滑动窗口
            'seasonal': self._seasonal_update,  # 季节性更新
            'event_driven': self._event_driven_update  # 事件驱动
        }
    
    def update_baseline(self, current_baseline, new_data, mode='sliding_window'):
        """
        根据模式更新基线
        """
        # 数据质量检查
        if not self._validate_data_quality(new_data):
            return current_baseline  # 数据质量差，不更新
        
        # 异常检测 - 防止异常数据污染基线
        if self._contains_anomaly(new_data, current_baseline):
            # 标记但不更新
            return self._mark_pending_review(current_baseline, new_data)
        
        # 选择更新策略
        update_func = self.update_modes.get(mode, self._sliding_window_update)
        
        # 执行更新
        new_baseline = update_func(current_baseline, new_data)
        
        # 平滑处理 - 防止基线突变
        smoothed_baseline = self._smooth_transition(
            current_baseline, 
            new_baseline,
            alpha=0.3  # 平滑系数
        )
        
        return smoothed_baseline
    
    def _sliding_window_update(self, baseline, new_data, window_days=90):
        """
        滑动窗口更新 - 最常用
        保留最近N天的数据计算基线
        """
        # 获取历史数据
        historical = baseline.get('historical_data', [])
        
        # 添加新数据
        historical.extend(new_data)
        
        # 保留窗口内数据
        cutoff_date = datetime.now() - timedelta(days=window_days)
        windowed_data = [d for d in historical if d['date'] > cutoff_date]
        
        # 重新计算基线
        calculator = BaselineCalculator(baseline['user_profile'])
        new_baseline = calculator.calculate_baseline(windowed_data)
        
        # 保留历史数据引用
        new_baseline['historical_data'] = windowed_data
        
        return new_baseline
```

---

## Prompt 2: 生产级长期行为分析

### 系统角色
你是一个专业的安全行为分析系统，基于统计学方法和机器学习算法分析用户长期行为模式。你需要识别细微的行为变化、预测潜在风险，并提供可操作的建议。

### 核心分析算法

```python
class ProductionAnomalyDetector:
    """
    生产级异常检测器
    组合多种算法，提高检测准确率
    """
    
    def __init__(self):
        self.algorithms = {
            'statistical': StatisticalDetector(),      # 统计方法
            'isolation_forest': IsolationForest(),     # 孤立森林
            'lstm_autoencoder': LSTMAutoencoder(),     # 深度学习
            'markov_chain': MarkovChainDetector(),     # 马尔可夫链
            'ensemble': EnsembleDetector()             # 集成方法
        }
    
    def detect_anomalies(self, features, baseline, user_profile):
        """
        多算法投票机制
        """
        results = {}
        weights = self._get_algorithm_weights(user_profile)
        
        # 运行各个检测器
        for name, detector in self.algorithms.items():
            try:
                result = detector.detect(features, baseline)
                results[name] = result
            except Exception as e:
                # 算法失败不影响整体
                results[name] = {'score': 0, 'confidence': 0}
        
        # 加权投票
        final_score = self._weighted_voting(results, weights)
        
        # 生成解释
        explanation = self._generate_explanation(results, final_score)
        
        return {
            'anomaly_score': final_score,
            'risk_level': self._score_to_risk_level(final_score),
            'algorithm_results': results,
            'explanation': explanation,
            'confidence': self._calculate_confidence(results)
        }
    
    def _get_algorithm_weights(self, user_profile):
        """
        根据用户类型动态调整算法权重
        """
        if user_profile['role'] in ['开发', '运维']:
            # 技术岗位行为多样，使用深度学习
            return {
                'statistical': 0.1,
                'isolation_forest': 0.2,
                'lstm_autoencoder': 0.4,
                'markov_chain': 0.1,
                'ensemble': 0.2
            }
        elif user_profile['role'] in ['财务', '人事']:
            # 规律性强，使用统计方法
            return {
                'statistical': 0.4,
                'isolation_forest': 0.2,
                'lstm_autoencoder': 0.1,
                'markov_chain': 0.2,
                'ensemble': 0.1
            }
        else:
            # 默认均衡权重
            return {name: 0.2 for name in self.algorithms}
```

### 生产输出格式

```json
{
  "analysis_id": "uuid-xxx",
  "timestamp": "2024-03-15T10:30:00Z",
  "user_id": "用户ID",
  "analysis_period": "2024-02-15 to 2024-03-15",
  
  "executive_summary": {
    "risk_score": 7.5,
    "risk_level": "HIGH",
    "confidence": 0.85,
    "key_findings": [
      "检测到数据外泄行为模式",
      "行为模式在最近7天发生显著变化"
    ],
    "recommended_action": "IMMEDIATE_REVIEW"
  },
  
  "detailed_analysis": {
    "pattern_analysis": {
      "baseline_deviation": {
        "temporal": 2.3,
        "access": 3.5,
        "volume": 4.2,
        "overall": 3.3
      },
      "trend_analysis": {
        "trend_type": "escalating",
        "change_point": "2024-03-08",
        "trend_confidence": 0.78
      },
      "periodicity_break": {
        "detected": true,
        "previous_pattern": "daily_9to6",
        "current_pattern": "irregular_late_night",
        "break_date": "2024-03-08"
      }
    },
    
    "anomaly_details": [
      {
        "date": "2024-03-10",
        "anomaly_type": "temporal",
        "description": "深夜2-4点持续活动",
        "statistical_significance": 0.001,  // p-value
        "algorithms_triggered": ["statistical", "lstm", "markov"],
        "context": "周末深夜，无加班记录"
      },
      {
        "date": "2024-03-12",
        "anomaly_type": "volume",
        "description": "下载500MB敏感数据",
        "statistical_significance": 0.0001,
        "algorithms_triggered": ["all"],
        "context": "超过日常均值50倍"
      }
    ],
    
    "risk_indicators": {
      "data_exfiltration": {
        "probability": 0.82,
        "evidence": [
          "大量数据下载",
          "访问多个敏感系统",
          "非常规时间活动"
        ],
        "pattern_match": "INSIDER_THREAT_PATTERN_3"
      },
      "account_compromise": {
        "probability": 0.35,
        "evidence": [
          "IP地址变化",
          "行为模式突变"
        ],
        "pattern_match": "COMPROMISED_ACCOUNT_PATTERN_1"
      }
    },
    
    "peer_comparison": {
      "peer_group": "同部门同岗位10人",
      "deviation_from_peers": 3.8,  // 标准差
      "percentile": 98,  // 在同组中的百分位
      "is_outlier": true
    }
  },
  
  "technical_details": {
    "algorithms_used": {
      "statistical": {"score": 8.2, "confidence": 0.9},
      "isolation_forest": {"score": 7.8, "confidence": 0.85},
      "lstm_autoencoder": {"score": 7.1, "confidence": 0.75},
      "markov_chain": {"score": 6.9, "confidence": 0.7},
      "ensemble": {"score": 7.5, "confidence": 0.88}
    },
    "feature_importance": {
      "download_volume": 0.35,
      "night_activity": 0.28,
      "system_diversity": 0.22,
      "login_frequency": 0.15
    },
    "model_metadata": {
      "baseline_last_updated": "2024-02-15",
      "baseline_sample_size": 90,
      "data_quality_score": 0.92,
      "processing_time_ms": 1250
    }
  },
  
  "recommendations": {
    "immediate": [
      {
        "action": "BLOCK_DOWNLOADS",
        "priority": "CRITICAL",
        "reason": "防止进一步数据外泄"
      },
      {
        "action": "FORCE_MFA",
        "priority": "HIGH",
        "reason": "确认用户身份"
      }
    ],
    "investigation": [
      "审查3月10-12日所有下载文件",
      "联系用户确认深夜活动原因",
      "检查是否有离职倾向"
    ],
    "long_term": [
      "实施DLP策略",
      "加强敏感数据访问审计",
      "定期安全意识培训"
    ]
  },
  
  "compliance_notes": {
    "gdpr_compliant": true,
    "data_retention_days": 90,
    "audit_trail": "analysis_log_20240315_xxx"
  }
}
```

### 处理特殊场景

#### 场景1：季节性业务（如财务月末结算）
```python
def handle_seasonal_pattern(self, features, user_profile):
    if user_profile['department'] == '财务' and is_month_end():
        # 调整阈值
        adjusted_threshold = baseline_threshold * 2.0
        # 标记为预期行为
        features['expected_peak'] = True
```

#### 场景2：组织变更（部门调整、权限变更）
```python
def handle_org_change(self, features, change_event):
    if change_event['type'] == 'role_change':
        # 重置基线
        return {'action': 'reset_baseline', 'grace_period_days': 30}
```

#### 场景3：批量异常（如系统故障导致）
```python
def handle_system_anomaly(self, anomalies):
    if len(anomalies) > threshold and all_same_pattern(anomalies):
        return {'type': 'system_issue', 'individual_risk': 'low'}
```

## 生产部署建议

### 1. 性能优化
```python
# 使用缓存减少重复计算
@lru_cache(maxsize=1000)
def calculate_baseline_cached(user_id, date_range):
    return calculate_baseline(user_id, date_range)

# 批处理提高效率
def batch_process_users(user_ids, batch_size=100):
    for batch in chunks(user_ids, batch_size):
        process_parallel(batch)
```

### 2. 监控指标
```python
metrics = {
    'false_positive_rate': monitor.track('fp_rate'),
    'detection_latency': monitor.track('latency_ms'),
    'baseline_drift': monitor.track('baseline_change'),
    'algorithm_agreement': monitor.track('algorithm_correlation')
}
```

### 3. A/B测试框架
```python
def ab_test_new_algorithm(user_id):
    if hash(user_id) % 100 < 10:  # 10%流量
        return new_algorithm.detect(user_id)
    else:
        return current_algorithm.detect(user_id)
```

## 总结

这个生产级系统的核心特点：

1. **鲁棒性**：处理各种数据质量问题
2. **自适应**：基线和阈值动态调整
3. **多算法融合**：提高准确率
4. **可解释性**：详细的证据链
5. **实时性**：支持大规模并发
6. **可扩展**：易于添加新算法和规则
