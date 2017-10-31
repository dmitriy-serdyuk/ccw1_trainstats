# Visualization of Graphs

## Training Statistics

Here is an example that I used for visualizing machine translation system training statistics, you can see them all here at [train.py](https://github.com/dendisuhubdy/ccw1_trainstats/blob/master/pytorch/non-gans/machinetranslation/train.py)

At first I defined which server that I'd want to stream to

```
from visdom import Visdom
viz = Visdom(server='http://suhubdy.com', port=51401)
```

then at the main training loop, which usually looks like this

```
def train_model(model, train_data, valid_data, fields, optim):

    train_iter = make_train_data_iter(train_data, opt)
    valid_iter = make_valid_data_iter(valid_data, opt)

    train_loss = make_loss_compute(model, fields["tgt"].vocab,
                                   train_data, opt)
    valid_loss = make_loss_compute(model, fields["tgt"].vocab,
                                   valid_data, opt)

    trunc_size = opt.truncated_decoder  # Badly named...
    shard_size = opt.max_generator_batches

    trainer = onmt.Trainer(model, train_iter, valid_iter,
                           train_loss, valid_loss, optim,
                           trunc_size, shard_size)

    for epoch in range(opt.start_epoch, opt.epochs + 1):
        print('')

        # 1. Train for one epoch on the training set.
        train_stats = trainer.train(epoch, report_func)
        print('Train perplexity: %g' % train_stats.ppl())
        print('Train accuracy: %g' % train_stats.accuracy())

        epochs_train_list.append(epoch)
        perplexity_train_list.append(train_stats.ppl())
        accuracy_train_list.append(train_stats.accuracy())

        # 2. Validate on the validation set.
        valid_stats = trainer.validate()
        print('Validation perplexity: %g' % valid_stats.ppl())
        print('Validation accuracy: %g' % valid_stats.accuracy())

        # 3. Log to remote server.
        if opt.exp_host:
            train_stats.log("train", experiment, optim.lr)
            valid_stats.log("valid", experiment, optim.lr)

        # 4. Update the learning rate
        trainer.epoch_step(valid_stats.ppl(), epoch)

        # 5. Drop a checkpoint if needed.
        if epoch >= opt.start_checkpoint_at:
            trainer.drop_checkpoint(opt, epoch, fields, valid_stats)

```

Let's do a step by step changes to this code to implement the training statistics visualization

### First step: define your Vizdom window

```
    # start Visdom visualization
    win_train_perplexity = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Training Perplexity of Kappa Experiment', caption='Perplexity.')
    )

    win_train_accuracy = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Training Accuracy of Kappa Experiment', caption='Accuracy')
    )

    win_valid_perplexity = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Validation Set Perplexity of Kappa Experiment', caption='Perplexity.')
    )

    win_valid_accuracy = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Validation Set Accuracy of Kappa Experiment', caption='Accuracy')
    )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_train_perplexity,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_train_perplexity,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_valid_accuracy,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_valid_perplexity,
            update='append'
            )
```

Those variables starting with `win_` are window placeholders that would be updated on at training time.

### Second step: adding the `updateTrace` inside the training inner loop

```
		...
        perplexity_train = train_stats.ppl()
        accuracy_train = train_stats.accuracy()

        perplexity_valid = valid_stats.ppl()
        accuracy_valid = valid_stats.accuracy()

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([accuracy_train]),
                win=win_train_accuracy,
                )

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([accuracy_valid]),
                win=win_valid_accuracy,
                )


        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([perplexity_train]),
                win=win_train_perplexity,
                )

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([perplexity_valid]),
                win=win_valid_perplexity,
                )

```

as you could see what we do is update the window `win` which is our previous designated windows, and update them with new plots points of X and Y. The whole training script finally looks like this

```
def train_model(model, train_data, valid_data, fields, optim):

    train_iter = make_train_data_iter(train_data, opt)
    valid_iter = make_valid_data_iter(valid_data, opt)

    train_loss = make_loss_compute(model, fields["tgt"].vocab,
                                   train_data, opt)
    valid_loss = make_loss_compute(model, fields["tgt"].vocab,
                                   valid_data, opt)

    trunc_size = opt.truncated_decoder  # Badly named...
    shard_size = opt.max_generator_batches

    trainer = onmt.Trainer(model, train_iter, valid_iter,
                           train_loss, valid_loss, optim,
                           trunc_size, shard_size)

    # start Visdom visualization
    win_train_perplexity = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Training Perplexity of Kappa Experiment', caption='Perplexity.')
    )

    win_train_accuracy = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Training Accuracy of Kappa Experiment', caption='Accuracy')
    )

    win_valid_perplexity = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Validation Set Perplexity of Kappa Experiment', caption='Perplexity.')
    )

    win_valid_accuracy = viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            opts=dict(title='Validation Set Accuracy of Kappa Experiment', caption='Accuracy')
    )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_train_perplexity,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_train_perplexity,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_valid_accuracy,
            update='append'
            )

    viz.line(
            X=np.array([0]),
            Y=np.array([0]),
            win=win_valid_perplexity,
            update='append'
            )


    for epoch in range(opt.start_epoch, opt.epochs + 1):
        print('')

        # 1. Train for one epoch on the training set.
        train_stats = trainer.train(epoch, report_func)
        print('Train perplexity: %g' % train_stats.ppl())
        print('Train accuracy: %g' % train_stats.accuracy())
        perplexity_train = train_stats.ppl()
        accuracy_train = train_stats.accuracy()


        # 2. Validate on the validation set.
        valid_stats = trainer.validate()
        print('Validation perplexity: %g' % valid_stats.ppl())
        print('Validation accuracy: %g' % valid_stats.accuracy())
        epochs_valid_list.append(epoch)
        perplexity_valid = valid_stats.ppl()
        accuracy_valid = valid_stats.accuracy()

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([accuracy_train]),
                win=win_train_accuracy,
                )

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([accuracy_valid]),
                win=win_valid_accuracy,
                )


        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([perplexity_train]),
                win=win_train_perplexity,
                )

        viz.updateTrace(
                X=np.array([epoch]),
                Y=np.array([perplexity_valid]),
                win=win_valid_perplexity,
                )

        # 3. Log to remote server.
        if opt.exp_host:
            train_stats.log("train", experiment, optim.lr)
            valid_stats.log("valid", experiment, optim.lr)

        # 4. Update the learning rate
        trainer.epoch_step(valid_stats.ppl(), epoch)

        # 5. Drop a checkpoint if needed.
        if epoch >= opt.start_checkpoint_at:
            trainer.drop_checkpoint(opt, epoch, fields, valid_stats)

```