---
type: 지식
created: 2026-08-20
updated: 2026-08-20
tags: [CSharp, Transaction, MSSQL, SqlTransaction, TransactionScope]
aliases: [C# Transaction, C# DB 트랜잭션, 인메모리 트랜잭션]
source: ChatGPT 대화 정리
---

# C# 트랜잭션 처리와 인메모리 롤백 예제

## 요약

- 트랜잭션은 여러 작업을 하나의 단위로 묶어 전부 성공하면 `Commit`, 하나라도 실패하면 `Rollback`하는 처리 방식이다.
- C# 언어가 일반 객체의 필드, `List`, `Dictionary` 변경을 자동으로 되돌려 주는 것은 아니다.
- DB에서는 SQL Server가 트랜잭션 로그와 잠금 또는 버전 관리를 통해 원자성, 일관성, 격리성, 지속성을 제공한다.
- DB 연결 없이도 변경 전 상태나 취소 동작을 직접 저장하면 트랜잭션 개념을 재현할 수 있다. 다만 이것은 DB 트랜잭션과 동일한 내구성이나 격리성을 제공하지 않는다.
- 단일 SQL 연결에서는 `SqlTransaction`, 여러 트랜잭션 지원 리소스를 하나의 범위로 묶을 때는 `TransactionScope`를 검토한다.

## 내용

## 1. 트랜잭션의 기본 흐름

트랜잭션의 가장 단순한 흐름은 다음과 같다.

```text
Begin
→ 작업 1
→ 작업 2
→ 모든 작업 성공: Commit
→ 중간에 실패: Rollback
```

대표적인 계좌 이체를 예로 들면 다음 두 작업은 함께 성공하거나 함께 실패해야 한다.

```text
A 계좌에서 300 차감
B 계좌에 300 추가
```

첫 번째 작업만 반영되고 두 번째 작업이 실패하면 데이터가 깨진다. 트랜잭션은 이런 부분 성공 상태를 방지한다.

## 2. ACID

DB 트랜잭션은 일반적으로 다음 성질을 목표로 한다.

| 속성 | 의미 |
| --- | --- |
| Atomicity, 원자성 | 전부 성공하거나 전부 실패한다. |
| Consistency, 일관성 | 트랜잭션 전후로 데이터 규칙을 유지한다. |
| Isolation, 격리성 | 동시에 실행되는 트랜잭션의 간섭을 제어한다. |
| Durability, 지속성 | Commit된 결과는 장애가 발생해도 보존되어야 한다. |

DB 없이 만든 메모리 롤백 예제는 원자성의 일부 개념만 흉내 낸다. 프로세스가 종료되면 상태와 롤백 정보가 모두 사라지므로 지속성을 제공하지 않는다.

## 3. `SqlTransaction` 기본 패턴

다음은 `Microsoft.Data.SqlClient`를 사용하는 일반적인 형태다.

```csharp
using System.Data;
using Microsoft.Data.SqlClient;

public static async Task TransferAsync(
    string connectionString,
    long fromAccountId,
    long toAccountId,
    long amount,
    CancellationToken cancellationToken = default)
{
    if (amount <= 0)
        throw new ArgumentOutOfRangeException(nameof(amount));

    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync(cancellationToken);

    await using var transaction =
        (SqlTransaction)await connection.BeginTransactionAsync(
            IsolationLevel.ReadCommitted,
            cancellationToken);

    try
    {
        await using (var debitCommand = connection.CreateCommand())
        {
            debitCommand.Transaction = transaction;
            debitCommand.CommandText = """
                UPDATE dbo.Account
                SET Balance = Balance - @Amount
                WHERE AccountId = @AccountId
                  AND Balance >= @Amount;
                """;

            debitCommand.Parameters.Add("@Amount", SqlDbType.BigInt).Value = amount;
            debitCommand.Parameters.Add("@AccountId", SqlDbType.BigInt).Value = fromAccountId;

            int affectedRows = await debitCommand.ExecuteNonQueryAsync(cancellationToken);

            if (affectedRows != 1)
                throw new InvalidOperationException("출금 계좌가 없거나 잔액이 부족합니다.");
        }

        await using (var creditCommand = connection.CreateCommand())
        {
            creditCommand.Transaction = transaction;
            creditCommand.CommandText = """
                UPDATE dbo.Account
                SET Balance = Balance + @Amount
                WHERE AccountId = @AccountId;
                """;

            creditCommand.Parameters.Add("@Amount", SqlDbType.BigInt).Value = amount;
            creditCommand.Parameters.Add("@AccountId", SqlDbType.BigInt).Value = toAccountId;

            int affectedRows = await creditCommand.ExecuteNonQueryAsync(cancellationToken);

            if (affectedRows != 1)
                throw new InvalidOperationException("입금 계좌를 찾을 수 없습니다.");
        }

        await transaction.CommitAsync(cancellationToken);
    }
    catch
    {
        try
        {
            await transaction.RollbackAsync(CancellationToken.None);
        }
        catch (Exception rollbackException)
        {
            // 실제 서비스에서는 원래 예외와 Rollback 예외를 함께 기록한다.
            Console.Error.WriteLine($"Rollback 실패: {rollbackException}");
        }

        throw;
    }
}
```

핵심은 다음과 같다.

1. 연결을 연다.
2. 해당 연결에서 트랜잭션을 시작한다.
3. 트랜잭션에 참여하는 모든 `SqlCommand`에 같은 `SqlTransaction`을 지정한다.
4. 전부 성공한 뒤에만 `Commit`한다.
5. 예외가 발생하면 `Rollback`한 뒤 원래 예외를 다시 던진다.

## 4. 자주 하는 실수

### Command에 Transaction을 지정하지 않는다

트랜잭션이 시작된 연결에서 명령을 실행하더라도 다음 지정이 빠지면 의도한 트랜잭션에 참여하지 않거나 예외가 발생할 수 있다.

```csharp
command.Transaction = transaction;
```

### 서로 다른 Connection을 사용한다

하나의 `SqlTransaction`은 해당 트랜잭션을 시작한 `SqlConnection`에 속한다. 다른 연결에서 만든 명령에 연결할 수 없다.

### 중간에 Commit한다

작업 1 이후 Commit하고 작업 2를 실행하면 두 작업은 더 이상 하나의 원자적 단위가 아니다.

### 예외를 삼킨다

Rollback 후 원래 실패를 호출자에게 알리지 않으면 상위 로직이 성공으로 오해할 수 있다.

```csharp
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### Rollback은 항상 성공한다고 가정한다

연결 단절이나 서버 장애로 Rollback 자체도 실패할 수 있다. 원래 예외와 Rollback 예외를 모두 관찰할 수 있도록 로그를 남긴다.

### 너무 긴 트랜잭션을 만든다

트랜잭션 안에서 외부 API 호출, 긴 연산, 사용자 입력 대기 등을 하면 잠금 유지 시간이 길어지고 교착 상태 가능성이 커진다. DB에서 꼭 함께 처리해야 하는 작업만 짧게 묶는다.

## 5. `TransactionScope`

`TransactionScope`는 범위 안에서 트랜잭션을 지원하는 리소스가 자동으로 참여하도록 만드는 방식이다.

```csharp
using System.Transactions;

var options = new TransactionOptions
{
    IsolationLevel = System.Transactions.IsolationLevel.ReadCommitted,
    Timeout = TransactionManager.DefaultTimeout
};

using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    options,
    TransactionScopeAsyncFlowOption.Enabled);

await ExecuteDatabaseWorkAsync();
await ExecuteAnotherTransactionalWorkAsync();

scope.Complete();
```

`Complete()`가 호출되지 않으면 범위를 빠져나갈 때 Rollback된다.

주의할 점은 다음과 같다.

- `async` 코드에서는 `TransactionScopeAsyncFlowOption.Enabled`를 사용한다.
- 일반 C# 객체의 필드 변경은 자동 Rollback되지 않는다.
- 여러 DB 연결이나 리소스를 묶으면 환경에 따라 분산 트랜잭션으로 승격될 수 있다.
- 단일 SQL 연결만 다룬다면 `SqlTransaction`이 동작 범위가 더 명확하다.

## 6. DB 연결 없이 트랜잭션 개념 만들기

DB가 없으면 리소스 관리자가 상태를 자동 복구해 주지 않는다. 따라서 변경 전 상태 또는 취소 동작을 직접 기록해야 한다.

다음 예제는 Rollback 시 실행할 동작을 `Stack<Action>`에 쌓는다. 스택을 사용하는 이유는 변경의 역순으로 되돌려야 하기 때문이다.

```csharp
public sealed class MemoryTransaction : IDisposable
{
    private readonly Stack<Action> _rollbackActions = new();
    private TransactionState _state = TransactionState.Active;

    public void RegisterRollback(Action rollbackAction)
    {
        ArgumentNullException.ThrowIfNull(rollbackAction);
        EnsureActive();
        _rollbackActions.Push(rollbackAction);
    }

    public void Commit()
    {
        EnsureActive();

        _rollbackActions.Clear();
        _state = TransactionState.Committed;
    }

    public void Rollback()
    {
        if (_state != TransactionState.Active)
            return;

        List<Exception>? errors = null;

        while (_rollbackActions.Count > 0)
        {
            Action rollbackAction = _rollbackActions.Pop();

            try
            {
                rollbackAction();
            }
            catch (Exception exception)
            {
                errors ??= new List<Exception>();
                errors.Add(exception);
            }
        }

        _state = TransactionState.RolledBack;

        if (errors is { Count: > 0 })
            throw new AggregateException("하나 이상의 Rollback 작업이 실패했습니다.", errors);
    }

    public void Dispose()
    {
        if (_state == TransactionState.Active)
            Rollback();
    }

    private void EnsureActive()
    {
        if (_state != TransactionState.Active)
            throw new InvalidOperationException($"현재 트랜잭션 상태는 {_state}입니다.");
    }

    private enum TransactionState
    {
        Active,
        Committed,
        RolledBack
    }
}
```

## 7. 계좌 이체 인메모리 예제

```csharp
public sealed class Account
{
    public Account(string name, long balance)
    {
        Name = name;
        Balance = balance;
    }

    public string Name { get; }
    public long Balance { get; set; }

    public override string ToString() => $"{Name}: {Balance}";
}
```

이체 로직은 값을 변경하기 전에 기존 상태를 저장하고 Rollback 동작을 먼저 등록한다.

```csharp
public static class BankService
{
    public static void Transfer(
        MemoryTransaction transaction,
        Account from,
        Account to,
        long amount)
    {
        ArgumentNullException.ThrowIfNull(transaction);
        ArgumentNullException.ThrowIfNull(from);
        ArgumentNullException.ThrowIfNull(to);

        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount));

        long beforeFromBalance = from.Balance;
        transaction.RegisterRollback(() => from.Balance = beforeFromBalance);

        from.Balance -= amount;

        if (from.Balance < 0)
            throw new InvalidOperationException("잔액이 부족합니다.");

        long beforeToBalance = to.Balance;
        transaction.RegisterRollback(() => to.Balance = beforeToBalance);

        to.Balance += amount;
    }
}
```

성공 예제:

```csharp
var accountA = new Account("A", 1_000);
var accountB = new Account("B", 500);

using (var transaction = new MemoryTransaction())
{
    BankService.Transfer(transaction, accountA, accountB, 300);
    transaction.Commit();
}

Console.WriteLine(accountA); // A: 700
Console.WriteLine(accountB); // B: 800
```

실패 예제:

```csharp
var accountA = new Account("A", 1_000);
var accountB = new Account("B", 500);

try
{
    using var transaction = new MemoryTransaction();

    BankService.Transfer(transaction, accountA, accountB, 3_000);
    transaction.Commit();
}
catch (Exception exception)
{
    Console.WriteLine(exception.Message);
}

Console.WriteLine(accountA); // A: 1000
Console.WriteLine(accountB); // B: 500
```

`Transfer` 도중 예외가 발생하면 `using`의 `Dispose()`가 호출되고 등록된 Rollback 동작이 실행된다.

## 8. 인메모리 예제의 한계

이 예제는 트랜잭션 개념을 학습하기 위한 코드다. 실제 데이터 보호 수단으로 DB 트랜잭션을 대체하지 못한다.

| 한계 | 설명 |
| --- | --- |
| 내구성 없음 | 프로세스가 종료되면 상태와 Rollback 정보가 사라진다. |
| 격리성 없음 | 다른 스레드가 같은 객체를 읽거나 수정하는 것을 막지 않는다. |
| Rollback 실패 가능 | 복구 동작 자체가 예외를 던질 수 있다. |
| 외부 부작용 복구 어려움 | 이미 전송한 이메일, 네트워크 요청, 결제 요청 등은 단순 대입으로 취소할 수 없다. |
| 크래시 복구 불가 | Commit 전 프로세스가 죽어도 자동 복구할 로그가 없다. |

실제 서비스에서 DB 외부 작업까지 묶어야 한다면 다음 패턴을 검토한다.

- 상태를 먼저 저장하고 비동기 후속 작업을 수행하는 Outbox 패턴
- 실패한 외부 작업을 반대 작업으로 보상하는 Saga 또는 보상 트랜잭션
- 멱등성 키를 사용한 중복 요청 방지
- 이벤트 로그를 기준으로 상태를 재구성하는 방식

## 9. 어떤 방식을 선택할까

| 상황 | 권장 방식 |
| --- | --- |
| 한 SQL Connection 안의 여러 쿼리 | `SqlTransaction` |
| 여러 트랜잭션 지원 리소스를 범위로 묶음 | `TransactionScope` 검토 |
| 단순 객체 변경을 되돌리는 테스트·학습 | 상태 복사 또는 Undo Stack |
| 외부 API와 DB를 함께 처리 | Outbox, Saga, 보상 처리 |
| 프로세스 장애 후에도 복구 필요 | DB나 영속 로그 사용 |

## 관련 문서

- [[MSSQL 로컬 개발 DB 구성과 메모리 최적화 설정]]
- [[지식_인덱스]]
