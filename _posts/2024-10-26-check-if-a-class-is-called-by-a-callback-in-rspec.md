---
layout: post
title: 'Check if a class is called by a callback in Rspec'
date: 2024-10-26 06:50:00 +0700
categories: dev
---

Given we have a fund service and a deposit model. When the deposit model change status from unpaid to paid, a lifecycle callback will call the fund service to move the fund. We need to test this callback

Normally when initialize the instance using `instance_double` and assert on it easily because the instance is the param of the tested method. However, when testing the callback, the callback will initialize the instance by itself so we need to listen on the class instead of the instance.

We have 2 ways to check if a fund service class is called when the callback is triggered

The first way, we use combo `allow` to spy the fund service class before triggering the callback. After the callback is triggered, we `expect` it with `to have_received`

```ruby
it 'spy first first and assert later' do
  fund_service = instance_double(FundService)

  allow(FundService).to receive(:new).and_return(fund_service)
  allow(fund_service).to receive(:move_fund)

  deposit_model.change_status_from_unpaid_to_paid # this will trigger fund moving callback

  expect(FundService).to have_received(:new)
  expect(fund_service).to have_received(:move_fund)
end
```

The second way, we spy and assert as the same time by using `expect` with `to receive`. This time, we assert before trigger callback.

```ruby
it 'spy and assert at the same time' do
  fund_service = instance_double(FundService)

  expect(FundService).to receive(:new)
  expect(fund_service).to receive(:move_fund)

  deposit_model.change_status_from_unpaid_to_paid # this will trigger fund moving callback
end
```

## Takeaways

- `allow` is for listen (spy) the class, instance
- `expect` is for listen (spy) and assert the class, instance at the same time
- Use `expect to receive` if you want to assert before the execution
- Use `expect to have_received` if you want to assert after the execution
