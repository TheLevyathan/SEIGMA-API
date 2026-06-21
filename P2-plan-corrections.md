# P2 — Corrections structurelles

Plan sauvegardé le 2026-06-20.
**À exécuter après que l'administrateur a créé des enregistrements pour les 47 modèles vides.**

---

## P2-1 — 47 modèles vides

Modèles sans aucun enregistrement. Une fois des données créées, relancer le script d'introspection pour peupler leurs schémas.

```
Carrier, Color, CompanyAddress, CustomerProfile, CustomerUnit, Employee,
EquipmentCategory, Expense, ExpenseCategory, Gender, Holiday,
InventoryLocation, InventoryLog, InventoryMove, InventoryStock,
LeadStatusReason, MailCopy, Payment, PaymentTerminal, PriceList,
ProductClass, ProductCost, ProductPrice, PurchaseAccountingMapping,
PurchaseInvoice, PurchaseInvoiceStatusReason, PurchaseOrder,
PurchaseOrderStatusReason, ReadOnlyRange, Receiving,
SalesOrderCustomList2, SalesOrderCustomList3, SalesProfit,
Shipping, ShippingMethod, Size, Subscription, Supplier,
SupplierClass, SupplierContact, SupplierDiscount, UnitBrand,
UnitCategory, UnitClass, UnitOfMeasure, UnitType, WarehouseProduct
```

---

## P2-3 — CreatedById / ModifiedById sans $ref

L'API confirme 3 structures constantes pour tous les champs liés :

- **UserReference** = `{UserId (string), Display (string)}`
- **ModelReference** = `{ReferenceId (string), Display (string), ModelCode (string)}`  
- **StatusReference** = `{ReferenceId, Display, ModelCode, Color}`

Plan :
1. Créer 3 schémas dans `components/schemas`
2. Remplacer `type: object` → `$ref` dans ~107 schémas
3. ~280 occurrences à patcher

---

## P2-5 — Cohérence architecturale

Consensus industrie (OpenAPI, Speakeasy, APIMatic) : `$ref` avec schémas nommés > objets inline.

Plan B→A :
1. Créer `UserReference`, `ModelReference`, `StatusReference`
2. Remplacer tous les `type: object` par `$ref` (~280 occurrences)
3. Aligner le schéma Activity

---

## P2-6 — Schémas d'erreur

Créer `ApiError` : `{message: string, code: integer}`
Ajouter à toutes les réponses 401/404 (~600 occurrences)

---

## P2-7 — operationIds

Renommer :
- `postAuthentification` → `authenticate`
- `createActivity` → garder (exception documentée)
- `postActivity` → `getActivitiesForDate`
- `postTimelog` → `addTimelog`

---

## Procédure de reprise

1. Créer les 47 enregistrements
2. Relancer `collect_seigma_schemas.py` pour obtenir les schémas
3. Exécuter P2-3 + P2-5 + P2-6 + P2-7
4. Régénérer Swagger + Redoc + PDF
5. Copier sur le partage

**Question en suspens :** Garder `ModelCode` dans `ModelReference` ?
