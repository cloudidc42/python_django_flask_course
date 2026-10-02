# Part 104: Machine Learning with Python

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ scikit-learn สำหรับ ML basics
- จัดการ data ด้วย pandas และ numpy
- Train และ evaluate models
- Serve ML models ด้วย FastAPI
- Save และ load models ด้วย joblib/pickle
- Deploy ML APIs สำหรับ production

---

## 1. Data Preparation ด้วย Pandas และ NumPy

```python
# ml/data_preparation.py
"""
Data Preparation สำหรับ Machine Learning
"""
import pandas as pd
import numpy as np
from sklearn.preprocessing import (
    StandardScaler, MinMaxScaler, LabelEncoder,
    OneHotEncoder, RobustScaler
)
from sklearn.impute import SimpleImputer
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from typing import Tuple, List, Optional


class DataPreparer:
    """จัดเตรียมข้อมูลสำหรับ ML"""
    
    def __init__(self, df: pd.DataFrame):
        self.df = df.copy()
        self.original_shape = df.shape
    
    def overview(self) -> dict:
        """สรุปข้อมูลเบื้องต้น"""
        return {
            "shape": self.df.shape,
            "dtypes": self.df.dtypes.to_dict(),
            "missing_values": self.df.isnull().sum().to_dict(),
            "missing_percentage": (self.df.isnull().sum() / len(self.df) * 100).to_dict(),
            "duplicates": self.df.duplicated().sum(),
            "memory_usage_mb": self.df.memory_usage(deep=True).sum() / 1024**2
        }
    
    def handle_missing_values(
        self,
        numeric_strategy: str = "median",
        categorical_strategy: str = "most_frequent",
        drop_threshold: float = 0.5
    ) -> pd.DataFrame:
        """
        จัดการ missing values
        
        Args:
            numeric_strategy: mean/median/most_frequent/constant
            categorical_strategy: most_frequent/constant
            drop_threshold: ลบ column ถ้ามี missing เกิน threshold
        """
        # ลบ columns ที่มี missing มากเกินไป
        missing_ratio = self.df.isnull().sum() / len(self.df)
        cols_to_drop = missing_ratio[missing_ratio > drop_threshold].index
        self.df.drop(columns=cols_to_drop, inplace=True)
        
        if cols_to_drop.tolist():
            print(f"Dropped columns with >{drop_threshold*100}% missing: {list(cols_to_drop)}")
        
        # Fill missing values
        numeric_cols = self.df.select_dtypes(include=[np.number]).columns
        categorical_cols = self.df.select_dtypes(include=["object", "category"]).columns
        
        if len(numeric_cols) > 0:
            imputer = SimpleImputer(strategy=numeric_strategy)
            self.df[numeric_cols] = imputer.fit_transform(self.df[numeric_cols])
        
        if len(categorical_cols) > 0:
            imputer = SimpleImputer(strategy=categorical_strategy, fill_value="Unknown")
            self.df[categorical_cols] = imputer.fit_transform(self.df[categorical_cols])
        
        return self.df
    
    def handle_outliers(
        self,
        columns: List[str] = None,
        method: str = "iqr",
        threshold: float = 1.5
    ) -> pd.DataFrame:
        """
        จัดการ outliers
        
        Methods:
        - iqr: Interquartile Range
        - zscore: Z-Score
        - clip: Clip ค่าที่เกิน boundaries
        """
        if columns is None:
            columns = self.df.select_dtypes(include=[np.number]).columns.tolist()
        
        for col in columns:
            if method == "iqr":
                Q1 = self.df[col].quantile(0.25)
                Q3 = self.df[col].quantile(0.75)
                IQR = Q3 - Q1
                lower = Q1 - threshold * IQR
                upper = Q3 + threshold * IQR
                
                # แทนด้วย boundary values แทนการลบ
                self.df[col] = self.df[col].clip(lower=lower, upper=upper)
            
            elif method == "zscore":
                z_scores = np.abs((self.df[col] - self.df[col].mean()) / self.df[col].std())
                self.df = self.df[z_scores < threshold]
        
        return self.df
    
    def feature_engineering(self) -> pd.DataFrame:
        """
        Feature Engineering ตัวอย่าง
        """
        # ตัวอย่างสำหรับข้อมูล datetime
        if "created_at" in self.df.columns:
            self.df["created_at"] = pd.to_datetime(self.df["created_at"])
            self.df["hour"] = self.df["created_at"].dt.hour
            self.df["day_of_week"] = self.df["created_at"].dt.dayofweek
            self.df["month"] = self.df["created_at"].dt.month
            self.df["is_weekend"] = (self.df["day_of_week"] >= 5).astype(int)
        
        # Log transformation สำหรับ skewed numeric columns
        for col in self.df.select_dtypes(include=[np.number]).columns:
            skewness = self.df[col].skew()
            if abs(skewness) > 1:
                # ตรวจสอบว่าค่าเป็น positive ก่อน log
                if self.df[col].min() > 0:
                    self.df[f"{col}_log"] = np.log1p(self.df[col])
        
        return self.df


# ตัวอย่างการใช้งาน
def prepare_ecommerce_data() -> Tuple[pd.DataFrame, pd.Series]:
    """ตัวอย่าง: เตรียมข้อมูล e-commerce สำหรับ predict customer churn"""
    
    # สร้าง sample data
    np.random.seed(42)
    n_samples = 1000
    
    data = {
        "user_id": range(n_samples),
        "age": np.random.randint(18, 70, n_samples),
        "total_orders": np.random.randint(0, 50, n_samples),
        "avg_order_value": np.random.exponential(500, n_samples),
        "days_since_last_order": np.random.randint(0, 365, n_samples),
        "total_spent": np.random.exponential(5000, n_samples),
        "email_open_rate": np.random.uniform(0, 1, n_samples),
        "customer_segment": np.random.choice(["bronze", "silver", "gold"], n_samples),
        "has_app": np.random.choice([0, 1], n_samples),
        # Target: 1 = churned, 0 = active
        "churned": (np.random.uniform(0, 1, n_samples) > 0.7).astype(int)
    }
    
    # เพิ่ม missing values เพื่อจำลอง real data
    df = pd.DataFrame(data)
    df.loc[np.random.choice(df.index, 50), "age"] = np.nan
    df.loc[np.random.choice(df.index, 30), "email_open_rate"] = np.nan
    
    target = df.pop("churned")
    df.drop("user_id", axis=1, inplace=True)
    
    return df, target
```

---

## 2. Model Training ด้วย scikit-learn

```python
# ml/model_training.py
"""
Model Training และ Evaluation
"""
from sklearn.model_selection import (
    train_test_split, cross_val_score,
    GridSearchCV, RandomizedSearchCV,
    StratifiedKFold
)
from sklearn.ensemble import RandomForestClassifier, GradientBoostingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.metrics import (
    classification_report, confusion_matrix,
    roc_auc_score, roc_curve, precision_recall_curve,
    f1_score, accuracy_score
)
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from typing import Dict, Any, List
import joblib
import time


class ModelTrainer:
    """จัดการ model training และ evaluation"""
    
    def __init__(
        self,
        X: pd.DataFrame,
        y: pd.Series,
        test_size: float = 0.2,
        random_state: int = 42
    ):
        self.X = X
        self.y = y
        self.random_state = random_state
        
        # Split data
        self.X_train, self.X_test, self.y_train, self.y_test = train_test_split(
            X, y,
            test_size=test_size,
            random_state=random_state,
            stratify=y  # ให้ split เท่ากันทั้ง class
        )
        
        # ระบุประเภทของ features
        self.numeric_features = X.select_dtypes(include=[np.number]).columns.tolist()
        self.categorical_features = X.select_dtypes(
            include=["object", "category"]
        ).columns.tolist()
        
        self.best_model = None
        self.best_pipeline = None
    
    def create_preprocessing_pipeline(self) -> ColumnTransformer:
        """สร้าง preprocessing pipeline"""
        numeric_transformer = Pipeline(steps=[
            ("imputer", SimpleImputer(strategy="median")),
            ("scaler", StandardScaler())
        ])
        
        categorical_transformer = Pipeline(steps=[
            ("imputer", SimpleImputer(strategy="most_frequent")),
            ("encoder", OneHotEncoder(handle_unknown="ignore", sparse_output=False))
        ])
        
        preprocessor = ColumnTransformer(transformers=[
            ("num", numeric_transformer, self.numeric_features),
            ("cat", categorical_transformer, self.categorical_features)
        ])
        
        return preprocessor
    
    def train_and_evaluate(
        self,
        models: Dict[str, Any] = None
    ) -> Dict[str, dict]:
        """Train และ compare หลาย models"""
        
        if models is None:
            models = {
                "Logistic Regression": LogisticRegression(
                    max_iter=1000,
                    random_state=self.random_state
                ),
                "Random Forest": RandomForestClassifier(
                    n_estimators=100,
                    random_state=self.random_state
                ),
                "Gradient Boosting": GradientBoostingClassifier(
                    n_estimators=100,
                    random_state=self.random_state
                )
            }
        
        preprocessor = self.create_preprocessing_pipeline()
        results = {}
        
        for name, model in models.items():
            print(f"\nTraining {name}...")
            start_time = time.time()
            
            # สร้าง full pipeline
            pipeline = Pipeline(steps=[
                ("preprocessor", preprocessor),
                ("classifier", model)
            ])
            
            # Cross-validation
            cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=self.random_state)
            cv_scores = cross_val_score(
                pipeline, self.X_train, self.y_train,
                cv=cv, scoring="roc_auc"
            )
            
            # Train final model
            pipeline.fit(self.X_train, self.y_train)
            
            # Evaluate on test set
            y_pred = pipeline.predict(self.X_test)
            y_prob = pipeline.predict_proba(self.X_test)[:, 1]
            
            training_time = time.time() - start_time
            
            results[name] = {
                "pipeline": pipeline,
                "cv_auc_mean": cv_scores.mean(),
                "cv_auc_std": cv_scores.std(),
                "test_auc": roc_auc_score(self.y_test, y_prob),
                "test_accuracy": accuracy_score(self.y_test, y_pred),
                "test_f1": f1_score(self.y_test, y_pred),
                "training_time": training_time,
                "classification_report": classification_report(
                    self.y_test, y_pred, output_dict=True
                )
            }
            
            print(f"  CV AUC: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")
            print(f"  Test AUC: {results[name]['test_auc']:.4f}")
            print(f"  Training time: {training_time:.2f}s")
        
        # หา best model
        best_name = max(results, key=lambda k: results[k]["test_auc"])
        self.best_model = results[best_name]["pipeline"]
        self.best_pipeline = self.best_model
        
        print(f"\n🏆 Best model: {best_name} (AUC: {results[best_name]['test_auc']:.4f})")
        return results
    
    def hyperparameter_tuning(
        self,
        param_grid: dict = None,
        n_iter: int = 20
    ) -> Pipeline:
        """Tune hyperparameters ด้วย RandomizedSearchCV"""
        
        if param_grid is None:
            param_grid = {
                "classifier__n_estimators": [100, 200, 300],
                "classifier__max_depth": [3, 5, 7, None],
                "classifier__min_samples_split": [2, 5, 10],
                "classifier__min_samples_leaf": [1, 2, 4]
            }
        
        preprocessor = self.create_preprocessing_pipeline()
        
        pipeline = Pipeline(steps=[
            ("preprocessor", preprocessor),
            ("classifier", RandomForestClassifier(random_state=self.random_state))
        ])
        
        cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=self.random_state)
        
        search = RandomizedSearchCV(
            pipeline,
            param_distributions=param_grid,
            n_iter=n_iter,
            cv=cv,
            scoring="roc_auc",
            random_state=self.random_state,
            n_jobs=-1,  # ใช้ทุก CPU cores
            verbose=1
        )
        
        print("Tuning hyperparameters...")
        search.fit(self.X_train, self.y_train)
        
        print(f"Best AUC: {search.best_score_:.4f}")
        print(f"Best params: {search.best_params_}")
        
        self.best_pipeline = search.best_estimator_
        return self.best_pipeline
    
    def get_feature_importance(self, top_n: int = 20) -> pd.DataFrame:
        """ดู feature importance"""
        if self.best_pipeline is None:
            raise ValueError("Train model first")
        
        preprocessor = self.best_pipeline.named_steps["preprocessor"]
        classifier = self.best_pipeline.named_steps["classifier"]
        
        # ดึง feature names หลัง preprocessing
        feature_names = []
        feature_names.extend(self.numeric_features)
        
        # One-hot encoded features
        if hasattr(preprocessor.named_transformers_["cat"], "named_steps"):
            encoder = preprocessor.named_transformers_["cat"].named_steps["encoder"]
            if hasattr(encoder, "get_feature_names_out"):
                cat_features = encoder.get_feature_names_out(self.categorical_features)
                feature_names.extend(cat_features)
        
        # Feature importance
        if hasattr(classifier, "feature_importances_"):
            importance = classifier.feature_importances_
            
            # Trim หรือ pad ให้ match
            min_len = min(len(feature_names), len(importance))
            feature_names = feature_names[:min_len]
            importance = importance[:min_len]
            
            importance_df = pd.DataFrame({
                "feature": feature_names,
                "importance": importance
            }).sort_values("importance", ascending=False).head(top_n)
            
            return importance_df
        
        return pd.DataFrame()
    
    def save_model(self, filepath: str):
        """บันทึก model"""
        if self.best_pipeline is None:
            raise ValueError("No model to save")
        
        model_data = {
            "pipeline": self.best_pipeline,
            "feature_names": self.X.columns.tolist(),
            "target_classes": self.y.unique().tolist(),
            "numeric_features": self.numeric_features,
            "categorical_features": self.categorical_features,
            "training_samples": len(self.X_train),
            "test_samples": len(self.X_test),
        }
        
        joblib.dump(model_data, filepath, compress=3)
        print(f"✅ Model saved to {filepath}")
```

---

## 3. Model Serving ด้วย FastAPI

```python
# ml/model_server.py
"""
Serve ML Model ผ่าน FastAPI
"""
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel, Field, validator
from typing import Optional, List, Dict, Any
import joblib
import numpy as np
import pandas as pd
import time
import os
import asyncio
from functools import lru_cache
import logging

logger = logging.getLogger(__name__)

app = FastAPI(
    title="Customer Churn Prediction API",
    description="Predict customer churn probability",
    version="1.0.0"
)


# ==================== Schemas ====================

class CustomerFeatures(BaseModel):
    """Input features สำหรับ prediction"""
    age: float = Field(..., ge=18, le=100, description="Customer age")
    total_orders: int = Field(..., ge=0, description="Total number of orders")
    avg_order_value: float = Field(..., ge=0, description="Average order value in THB")
    days_since_last_order: int = Field(..., ge=0, description="Days since last order")
    total_spent: float = Field(..., ge=0, description="Total amount spent in THB")
    email_open_rate: float = Field(..., ge=0, le=1, description="Email open rate (0-1)")
    customer_segment: str = Field(..., description="Customer segment: bronze/silver/gold")
    has_app: int = Field(..., ge=0, le=1, description="Has mobile app: 0/1")
    
    @validator("customer_segment")
    def validate_segment(cls, v):
        valid_segments = ["bronze", "silver", "gold"]
        if v.lower() not in valid_segments:
            raise ValueError(f"customer_segment must be one of: {valid_segments}")
        return v.lower()


class BatchPredictionRequest(BaseModel):
    """Batch prediction request"""
    customers: List[CustomerFeatures] = Field(..., min_items=1, max_items=1000)


class PredictionResponse(BaseModel):
    """Prediction response"""
    churn_probability: float
    churn_prediction: bool
    risk_level: str
    confidence: str
    prediction_time_ms: float


class BatchPredictionResponse(BaseModel):
    """Batch prediction response"""
    predictions: List[PredictionResponse]
    total_records: int
    high_risk_count: int
    prediction_time_ms: float


# ==================== Model Loading ====================

class ModelManager:
    """จัดการ ML model"""
    
    def __init__(self):
        self.model = None
        self.model_data = None
        self.model_version = None
        self.model_loaded_at = None
    
    def load_model(self, model_path: str):
        """โหลด model"""
        try:
            self.model_data = joblib.load(model_path)
            self.model = self.model_data["pipeline"]
            self.model_version = os.path.basename(model_path)
            self.model_loaded_at = time.time()
            logger.info(f"Model loaded: {model_path}")
        except Exception as e:
            logger.error(f"Failed to load model: {e}")
            raise
    
    def predict(self, features: dict) -> dict:
        """Predict จาก features dict"""
        if self.model is None:
            raise ValueError("Model not loaded")
        
        df = pd.DataFrame([features])
        
        # Predict
        probability = self.model.predict_proba(df)[0][1]
        prediction = bool(probability >= 0.5)
        
        # Risk level
        if probability >= 0.8:
            risk_level = "HIGH"
        elif probability >= 0.5:
            risk_level = "MEDIUM"
        else:
            risk_level = "LOW"
        
        # Confidence
        confidence_value = abs(probability - 0.5) * 2
        if confidence_value >= 0.7:
            confidence = "HIGH"
        elif confidence_value >= 0.4:
            confidence = "MEDIUM"
        else:
            confidence = "LOW"
        
        return {
            "churn_probability": round(float(probability), 4),
            "churn_prediction": prediction,
            "risk_level": risk_level,
            "confidence": confidence
        }
    
    def batch_predict(self, features_list: List[dict]) -> List[dict]:
        """Batch prediction"""
        if self.model is None:
            raise ValueError("Model not loaded")
        
        df = pd.DataFrame(features_list)
        probabilities = self.model.predict_proba(df)[:, 1]
        
        results = []
        for prob in probabilities:
            if prob >= 0.8:
                risk_level = "HIGH"
            elif prob >= 0.5:
                risk_level = "MEDIUM"
            else:
                risk_level = "LOW"
            
            confidence_value = abs(prob - 0.5) * 2
            confidence = "HIGH" if confidence_value >= 0.7 else "MEDIUM" if confidence_value >= 0.4 else "LOW"
            
            results.append({
                "churn_probability": round(float(prob), 4),
                "churn_prediction": bool(prob >= 0.5),
                "risk_level": risk_level,
                "confidence": confidence
            })
        
        return results


model_manager = ModelManager()

# โหลด model เมื่อเริ่ม server
MODEL_PATH = os.getenv("MODEL_PATH", "models/churn_model_v1.joblib")


@app.on_event("startup")
async def startup_event():
    """โหลด model เมื่อ server เริ่ม"""
    if os.path.exists(MODEL_PATH):
        try:
            model_manager.load_model(MODEL_PATH)
            logger.info("Model loaded successfully at startup")
        except Exception as e:
            logger.error(f"Failed to load model at startup: {e}")
    else:
        logger.warning(f"Model file not found: {MODEL_PATH}")


# ==================== Endpoints ====================

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "model_loaded": model_manager.model is not None,
        "model_version": model_manager.model_version
    }


@app.post("/predict", response_model=PredictionResponse)
async def predict_single(customer: CustomerFeatures):
    """ทำนายการ churn สำหรับ customer คนเดียว"""
    
    if model_manager.model is None:
        raise HTTPException(status_code=503, detail="Model not available")
    
    start_time = time.perf_counter()
    
    try:
        result = model_manager.predict(customer.model_dump())
        
        return PredictionResponse(
            **result,
            prediction_time_ms=round((time.perf_counter() - start_time) * 1000, 2)
        )
    except Exception as e:
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")


@app.post("/predict/batch", response_model=BatchPredictionResponse)
async def predict_batch(request: BatchPredictionRequest):
    """Batch prediction สำหรับหลาย customers"""
    
    if model_manager.model is None:
        raise HTTPException(status_code=503, detail="Model not available")
    
    start_time = time.perf_counter()
    
    try:
        features_list = [c.model_dump() for c in request.customers]
        predictions = model_manager.batch_predict(features_list)
        
        # สร้าง response objects
        prediction_responses = [
            PredictionResponse(
                **pred,
                prediction_time_ms=0
            )
            for pred in predictions
        ]
        
        high_risk_count = sum(1 for p in predictions if p["risk_level"] == "HIGH")
        
        return BatchPredictionResponse(
            predictions=prediction_responses,
            total_records=len(predictions),
            high_risk_count=high_risk_count,
            prediction_time_ms=round((time.perf_counter() - start_time) * 1000, 2)
        )
    except Exception as e:
        logger.error(f"Batch prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/model/reload")
async def reload_model():
    """Reload model (สำหรับ update model ใหม่)"""
    try:
        model_manager.load_model(MODEL_PATH)
        return {"status": "success", "message": "Model reloaded"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/model/info")
async def get_model_info():
    """ดูข้อมูล model ที่โหลดอยู่"""
    if model_manager.model is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    
    model_data = model_manager.model_data
    return {
        "version": model_manager.model_version,
        "loaded_at": model_manager.model_loaded_at,
        "feature_count": len(model_data.get("feature_names", [])),
        "training_samples": model_data.get("training_samples"),
        "features": model_data.get("feature_names", [])
    }
```

---

## 4. Model Saving และ Loading

```python
# ml/model_versioning.py
"""
Model Versioning และ Registry
"""
import joblib
import json
import os
import hashlib
import time
from pathlib import Path
from typing import Optional
import boto3


class ModelRegistry:
    """จัดการ model versions"""
    
    def __init__(self, models_dir: str = "models", s3_bucket: str = None):
        self.models_dir = Path(models_dir)
        self.models_dir.mkdir(exist_ok=True)
        self.s3_bucket = s3_bucket
        self.registry_file = self.models_dir / "registry.json"
        self._load_registry()
    
    def _load_registry(self):
        """โหลด registry"""
        if self.registry_file.exists():
            with open(self.registry_file, "r") as f:
                self.registry = json.load(f)
        else:
            self.registry = {"models": {}, "latest": {}}
    
    def _save_registry(self):
        """บันทึก registry"""
        with open(self.registry_file, "w") as f:
            json.dump(self.registry, f, indent=2)
    
    def save_model(
        self,
        model,
        model_name: str,
        metrics: dict,
        metadata: dict = None,
        promote_to_production: bool = False
    ) -> str:
        """บันทึก model พร้อม version"""
        
        # สร้าง version ID
        timestamp = int(time.time())
        version_id = f"v{timestamp}"
        
        # บันทึก model
        model_path = self.models_dir / f"{model_name}_{version_id}.joblib"
        
        model_data = {
            "model": model,
            "metrics": metrics,
            "metadata": metadata or {},
            "version": version_id,
            "created_at": timestamp,
            "model_name": model_name
        }
        
        joblib.dump(model_data, model_path, compress=3)
        
        # คำนวณ checksum
        with open(model_path, "rb") as f:
            checksum = hashlib.md5(f.read()).hexdigest()
        
        # Update registry
        if model_name not in self.registry["models"]:
            self.registry["models"][model_name] = {}
        
        self.registry["models"][model_name][version_id] = {
            "path": str(model_path),
            "metrics": metrics,
            "created_at": timestamp,
            "checksum": checksum,
            "status": "staging",
            "file_size_mb": model_path.stat().st_size / 1024**2
        }
        
        if promote_to_production:
            self.promote_to_production(model_name, version_id)
        else:
            self._save_registry()
        
        # อัพโหลดไปยัง S3 ถ้า configured
        if self.s3_bucket:
            self._upload_to_s3(model_path, model_name, version_id)
        
        print(f"✅ Model saved: {model_name} {version_id}")
        return version_id
    
    def load_model(
        self,
        model_name: str,
        version_id: str = None
    ) -> dict:
        """โหลด model"""
        if version_id is None:
            # โหลด production version
            version_id = self.registry["latest"].get(model_name)
            if not version_id:
                raise ValueError(f"No production model for: {model_name}")
        
        if (model_name not in self.registry["models"] or
                version_id not in self.registry["models"][model_name]):
            raise ValueError(f"Model not found: {model_name} {version_id}")
        
        model_info = self.registry["models"][model_name][version_id]
        model_path = model_info["path"]
        
        # ตรวจสอบ checksum
        with open(model_path, "rb") as f:
            current_checksum = hashlib.md5(f.read()).hexdigest()
        
        if current_checksum != model_info["checksum"]:
            raise ValueError(f"Model file corrupted: checksum mismatch")
        
        return joblib.load(model_path)
    
    def promote_to_production(self, model_name: str, version_id: str):
        """Promote model version ขึ้น production"""
        if model_name not in self.registry["models"]:
            raise ValueError(f"Model not found: {model_name}")
        
        if version_id not in self.registry["models"][model_name]:
            raise ValueError(f"Version not found: {version_id}")
        
        # ลด status ของ production เก่า
        old_prod = self.registry["latest"].get(model_name)
        if old_prod and old_prod in self.registry["models"][model_name]:
            self.registry["models"][model_name][old_prod]["status"] = "archived"
        
        # ตั้ง version ใหม่เป็น production
        self.registry["models"][model_name][version_id]["status"] = "production"
        self.registry["latest"][model_name] = version_id
        
        self._save_registry()
        print(f"✅ Promoted {model_name} {version_id} to production")
    
    def list_versions(self, model_name: str) -> list:
        """แสดง versions ทั้งหมด"""
        if model_name not in self.registry["models"]:
            return []
        
        versions = []
        for version_id, info in self.registry["models"][model_name].items():
            versions.append({
                "version": version_id,
                "status": info["status"],
                "metrics": info["metrics"],
                "created_at": info["created_at"],
                "is_latest": self.registry["latest"].get(model_name) == version_id
            })
        
        return sorted(versions, key=lambda x: x["created_at"], reverse=True)
    
    def _upload_to_s3(self, model_path: Path, model_name: str, version_id: str):
        """อัพโหลด model ไปยัง S3"""
        s3 = boto3.client("s3")
        s3_key = f"models/{model_name}/{version_id}/{model_path.name}"
        
        s3.upload_file(
            str(model_path),
            self.s3_bucket,
            s3_key
        )
        print(f"  Uploaded to s3://{self.s3_bucket}/{s3_key}")


# ตัวอย่าง training pipeline แบบสมบูรณ์
def run_training_pipeline():
    """Complete ML Training Pipeline"""
    
    print("🚀 Starting ML Training Pipeline...")
    
    # 1. Load Data
    print("\n1. Loading data...")
    X, y = prepare_ecommerce_data()
    
    # 2. Data Preparation
    print("\n2. Preparing data...")
    preparer = DataPreparer(X)
    print(preparer.overview())
    X = preparer.handle_missing_values()
    X = preparer.handle_outliers()
    
    # 3. Train Models
    print("\n3. Training models...")
    trainer = ModelTrainer(X, y)
    results = trainer.train_and_evaluate()
    
    # 4. Hyperparameter Tuning
    print("\n4. Tuning best model...")
    best_pipeline = trainer.hyperparameter_tuning(n_iter=10)
    
    # 5. Feature Importance
    print("\n5. Feature importance:")
    importance_df = trainer.get_feature_importance()
    print(importance_df.head(10).to_string())
    
    # 6. Save Model
    print("\n6. Saving model...")
    registry = ModelRegistry(models_dir="models")
    
    # ดึง metrics ของ best model
    best_model_name = max(results, key=lambda k: results[k]["test_auc"])
    metrics = {
        "test_auc": results[best_model_name]["test_auc"],
        "test_f1": results[best_model_name]["test_f1"],
        "test_accuracy": results[best_model_name]["test_accuracy"]
    }
    
    version_id = registry.save_model(
        model=best_pipeline,
        model_name="churn_prediction",
        metrics=metrics,
        metadata={
            "algorithm": best_model_name,
            "training_samples": len(trainer.X_train),
            "features": X.columns.tolist()
        },
        promote_to_production=True
    )
    
    print(f"\n✅ Training pipeline completed! Version: {version_id}")
    return version_id
```

---

## 5. สรุป Part 104

✅ จัดเตรียมข้อมูลด้วย pandas: handle missing values, outliers, feature engineering
✅ Train ML models ด้วย scikit-learn: Logistic Regression, Random Forest, Gradient Boosting
✅ Evaluate models ด้วย cross-validation, AUC, F1-score, confusion matrix
✅ Tune hyperparameters ด้วย RandomizedSearchCV
✅ Serve ML models ผ่าน FastAPI พร้อม batch prediction
✅ จัดการ model versioning และ registry พร้อม promotion to production
✅ เก็บ models ด้วย joblib พร้อม checksum verification

## ➡️ ถัดไป: Part 105 - Course Completion & Career Path

*Part 104/105 | Python Course - World-Class Level*
