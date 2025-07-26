# TelecomX: análisis de evaluación parte 2.
---

### Predicción de Churn (Cancelación de Clientes)

---

## Propósito del análisis
El objetivo central de este proyecto es **predecir la probabilidad de que un cliente cancele sus servicios de telecomunicaciones (churn)**.  
Con este modelo la empresa podrá:
- Identificar segmentos de alto riesgo y actuar **antes** de que el cliente abandone.  
- Optimizar campañas de retención y aumentar el **Lifetime Value (LTV)**.  
- Reducir costos de adquisición al disminuir la rotación.

---


---

## 🔧 Proceso de preparación de datos

### 1️⃣ Columnas y tipos
| **Variable**        | **Tipo**   | **Descripción** (ejemplo) |
|---------------------|------------|---------------------------|
| `customerID`        | ID         | Identificador único |
| `Churn`             | Binaria    | 0 = No canceló, 1 = Canceló |
| `gender`            | Categórica | Male / Female |
| `SeniorCitizen`     | Numérica   | 0 o 1 |
| `Partner`, `Dependents`, `PhoneService`, etc. | Categóricas | Sí/No |
| `Charges_Monthly`   | Numérica   | Cargo mensual en USD |
| `Cuentas_Diarias`   | Numérica   | Cargo promedio diario |

### 2️⃣ Codificación / normalización
- **Categóricas binarias**: Sí/No → 1/0  
- **Categóricas ordinales**:  
  - *Contract*: `Month-to-month=0`, `One year=1`, `Two year=2`  
  - *PaymentMethod*: `Mailed check=0`, `Electronic=1`, `Credit card=2`, `Bank transfer=3`  
- **Numéricas**: se mantienen en escala original (no se escalaron porque LightGBM es insensible a la escala).

### 3️⃣ División de datos
```python
X = df_final.drop(columns=['customerID', 'Churn'])
y = df_final['Churn']

from sklearn.model_selection import train_test_split
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=42, stratify=y
)
```
---
## Resultados
---

Evaluación de los modelos


---

Los modelos fueron entrenados en dos ciclos. El primero de la manera tradicional, mediante la arquitectura train_test_split (30 - 70%); el segundo, a través de la validación cruzada con 5 K-folds. El objetivo de esto era visualizar una mejoría en la predicción. Ante esto, se procede con la siguiente tabla de inferencia:

| Modelo                 | Acc. Split | Acc. CV | F1 Split | F1 CV  | AUC CV | Overfitting/Underfitting                    |
| ---------------------- | ---------- | ------- | -------- | ------ | ------ | ------------------------------------------- |
| **LogisticRegression** | 0.7643     | 0.7711  | 0.7745   | 0.7799 | 0.8520 | Underfitting (bajo desempeño global)        |
| **SVM**                | 0.8161     | 0.8208  | 0.8177   | 0.8115 | 0.9168 | Bien generalizado                           |
| **Decision Tree**      | 0.8119     | 0.8004  | 0.8133   | 0.7825 | 0.8008 | Ligeramente sobreajustado                   |
| **Random Forest**      | 0.8451     | 0.8355  | 0.8417   | 0.8041 | 0.9287 | Ligeramente sobreajustado                   |
| **Gradient Boosting**  | 0.8573     | 0.8438  | 0.8582   | 0.8238 | 0.9388 | Muy buen desempeño, generaliza bien         |
| **KNN**                | 0.7775     | 0.7892  | 0.7818   | 0.7883 | 0.8746 | Ligeramente simple (baja AUC)               |
| **XGBoost**            | 0.8499     | 0.8344  | 0.8481   | 0.8062 | 0.9328 | Ligeramente sobreajustado                   |
| **LightGBM**           | 0.8576     | 0.8385  | 0.8566   | 0.8093 | 0.9379 | Mejor modelo global (consistente y robusto) |



---

**¿Cuál fue el mejor modelo entonces?**

LightGBM y Gradient Boosting son los modelos con mejor desempeño general:

- Altos valores de precisión, recall y F1 en ambas estrategias.

- Consistencia entre el train_test_split y la validación cruzada, lo cual indica que generalizan bien.

- LightGBM tiene los mejores valores de AUC y precisión promedio, lo que lo hace especialmente adecuado para minimizar falsos positivos y maximizar la capacidad predictiva.

---

Bajo este panorama se propone lo siguiente:

| Modelo             | ¿Vale la pena seguirlo? | Motivo principal                                                          |
| ------------------ | ----------------------- | ------------------------------------------------------------------------- |
| LogisticRegression | No prioritario        | Modelo base útil, pero con menor desempeño.                               |
| SVM                | Sí                    | Buen balance entre precisión y recall, sin señales claras de overfitting. |
| Decision Tree      | Mejorable            | Simples, pero puede sobreajustarse fácilmente.                            |
| Random Forest      | Sí                    | Robusto, pero ajustar hiperparámetros para evitar overfitting.            |
| Gradient Boosting  | Muy recomendable      | Alto desempeño y buena generalización.                                    |
| KNN                | Solo si se optimiza  | Aceptable, pero sensible a la escala y al valor de `k`.                   |
| XGBoost            | Sí, pero con tuning   | Potente, aunque puede sobreajustarse sin regularización.                  |
| LightGBM           | **El mejor**         | Mejor equilibrio general. Altas métricas y buena generalización.          |



---



---
Interpretación del 2do mejor modelo (XGBoost)
---

| **Variable**                | **Impacto** | **Interpretación**                                                |
| --------------------------- | ----------- | ----------------------------------------------------------------- |
| **Tenure (meses)**          | 15.2%       | Clientes con < 12 meses tienen 3.2x más probabilidad de cancelar. |
| **Contract (tipo)**         | 12.8%       | Contratos *month-to-month* representan 78% de cancelaciones.      |
| **Charges\_Monthly**        | 11.4%       | Clientes con cargos > \$90/mes tienen 2.1x más riesgo.            |
| **PaymentMethod**           | 9.7%        | *Electronic check* (vs. automático) aumenta 1.8x la cancelación.  |
| **InternetService (Fiber)** | 8.9%        | Fibra óptica tiene 1.5x más cancelaciones vs. DSL.                |



---


---


Estrategias de Retención Recomendadas


---


1. Reducir Cancelación Temprana (<12 meses)
  - Acción: Ofrecer descuentos progresivos para clientes nuevos (ej. 20% descuento en mes 3-6).

  - Canal: Email/SMS automatizado al detectar inactividad.

2. Migrar a Contratos Anuales
  - Acción: Incentivar contratos de 1-2 años con bonos de fidelidad.
  - Impacto Esperado: Reducción en churn.

3. Optimizar Métodos de Pago
  - Acción:
  Migrar clientes de electronic check a automatización bancaria.
  - Ofrecer descuento 5% por pago automático.
  Impacto Esperado: Disminuir churn en 1.8x.

4. Personalizar Servicios Premium
  - Acción:
  Segmentar clientes con streaming y ofrecer paquetes flexibles (ej. "TV + Internet" con 15% descuento).
  - Alertas de uso excesivo para evitar "bill shock".

5. Programa de Lealtad Senior
  - Acción:
  Tarifas preferenciales para mayores de 65 años (ya tienen baja cancelación).
  - Soporte técnico prioritario.
