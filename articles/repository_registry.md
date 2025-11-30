---
title: "Design: Repository Registry and Transactions at the Service Level"
author: "Vadim Inshakov"
---

![](https://miro.medium.com/v2/resize:fit:1400/1*5QFMdnjspncc6XgMAhvVmQ.png)

Sometimes, after dividing an application into layers, we discover that the service/use-case layer becomes entangled with storage logic, especially when transactions are needed. This leads to poor application design, making it fragile. I want to demonstrate the most organic way of interaction between the service and repository layers in a transactional scenario — the transactional method of the repository registry.

## What is a Repository Registry?

Imagine you are working with a shopping cart, and you need two repositories: one for managing the cart and another for the items populating the cart.

```go
type BasketRepository interface {
    // GetByID returns basket by ID
    GetByID(id int64) (*aggregates.Basket, error)
    // Save saves basket
    Save(basket *aggregates.Basket) error
}

type ItemsRepository interface {
    // GetBasketItems returns items in basket
    GetByBasketID(id int64) (*vos.BasketItem, error)
    // Save saves one item in basket
    Save(basket *vos.BasketItem) error
}
```

These repositories are used in our hypothetical application in the service layer to add items to the shopping cart.

```go
type anemicBasketService struct {
    basketRepo repository.BasketRepository
    itemsRepo  repository.ItemsRepository
    producer   broker.Producer
}

// ...

for _, itemForSave := range basket.Items {
    if err := s.itemsRepo.Save(itemForSave); err != nil {
        return err
    }
}

if err := s.basketRepo.Save(basket); err != nil {
    return err
}
```

Let’s add another abstraction to this simple structure — a repository registry. It’s simply a structure that holds all the repositories of the application.

```go
type RepositoryRegistry interface {
    Basket() BasketRepository
    Items() ItemsRepository
}
```

This is the repository registry interface, responsible for storing and providing access to individual repositories.

Now, instead of directly accessing each repository in the service layer, we can use the repository registry. This makes it easier to work with various repositories and increases the level of abstraction in our code:

```go
type basketService struct {
    repo     repository.RepositoryRegistry
    producer broker.Producer
}

// ...

for _, itemForSave := range basket.Items {
    if err := s.repo.Items().Save(itemForSave); err != nil {
        return err
    }
}

if err := s.repo.Basket().Save(basket); err != nil {
    return err
}
```

Almost nothing has changed. Why do we need this abstraction? For more convenient transaction handling.

## Leaky transaction design

What if there is a failure while saving item positions, resulting in the items partially existing in the database but with no corresponding entries in the shopping cart?

```go
for _, itemForSave := range basket.Items {
    if err := s.repo.Items().Save(itemForSave); err != nil {
        return err
    }
}

// --- app crashes here! ---

if err := s.repo.Basket().Save(basket); err != nil {
    return err
}
```

We need a transaction. This is where problems begin, breaking down the boundaries between different layers of the application — the storage layer and the service layer.

You’ve probably seen in some projects services/usecases with transactional logic resembling this:

```go
func (s *txAnemicBasketService) AddItem(basketID int64, item vos.BasketItem) error {
    tx, err := s.txManager.Begin(context.Background())
    if err != nil {
        return err
    }

    basket, err := s.basketRepo.GetByID(tx, basketID)
    if err != nil {
        return err
    }

    // some manipulations with basket and item here ...

    for _, itemForSave := range basket.Items {
        if err := s.itemsRepo.Save(tx, itemForSave); err != nil {
            return err
        }
    }

    if err := s.basketRepo.Save(tx, basket); err != nil {
        return err
    }

    return tx.Commit()
}
```

This is a highly simplified code, the essence of which is that a database-specific transaction object is created at the service level (the service starts owning information it doesn’t actually need) and is passed down to repository methods. In practice, this approach leads to intertwining the complexities of business logic with storage specifics, making your code convoluted and fragile.

On the repository level, you end up duplicating methods with and without transactions. Either that or hiding transactions in a context, making repository code complex, the context heavy, and the transactional logic opaque.

If you want to put an end to this, let’s discuss how.

## Transactional Method in the Repository Registry

We need to add another method to our repository registry that will handle transactions.

```go
type RepositoryRegistry interface {
    Basket() BasketRepository
    Items() ItemsRepository
    Transaction(ctx context.Context, fn func(repo RepositoryRegistry) error) error
}
```

As you can see, we added the `Transaction` method, which takes a callback with an instance of the repository registry.

```go
func (r *RepoRegistry) Transaction(ctx context.Context, fn func(repo *RepoRegistry) error) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }

    newrepo, err := New(r.db, tx)
    if err != nil {
        return err
    }

    if err := fn(newrepo); err != nil {
        if err := tx.Rollback(); err != nil {
            return err
        }
        return err
    }

    if err := tx.Commit(); err != nil {
        return err
    }

    return err
}
```

The key point here is that we create a new master repository with an open transaction (`newrepo`), through which all subsequent operations of derived repositories will be executed. We then pass this object into the callback (`fn(newrepo)`). What do transactions at the service level look like now:

```go
// business logic ...

err = repo.Transaction(context.Background(), func(repo *RepoRegistry) error {
    if err := repo.Basket().Save(basket); err != nil {
        return err
    }
    if err := repo.Items().Save(items[index]); err != nil {
        return err
    }

    return nil
})

// send events ...
```

## What’s next?

Interested in more examples of good design? I’ve created a repository with simple examples at [https://github.com/vadiminshakov/dddgo](https://github.com/vadiminshakov/dddgo). The repository showcases widely accepted best practices and some anti-patterns for clarity. This project is constantly evolving, so stay tuned for updates or suggest changes, improvements, and additions.