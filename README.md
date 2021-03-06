[![GitHub Workflow Status](https://img.shields.io/github/workflow/status/olegtarasov/FastText.NetWrapper/Build%20and%20publish?style=flat-square)](https://github.com/olegtarasov/FastText.NetWrapper/actions)
[![Nuget](https://img.shields.io/nuget/v/FastText.NetWrapper?style=flat-square)](https://www.nuget.org/packages/FastText.NetWrapper)
[![Donwloads](https://img.shields.io/nuget/dt/FastText.NetWrapper?label=Nuget&style=flat-square)](https://www.nuget.org/packages/FastText.NetWrapper)

# FastText.NetWrapper

This is a cross-platform .NET Standard wrapper for Facebook's [FastText](https://github.com/facebookresearch/fastText) library. 
The wrapper comes with bundled precompiled native binaries for all three platforms: Windows, Linux and MacOs.

Just add it to your project and start using it! No additional setup required. This library will unpack and call appropriate native 
binary depending on target platform.

## Usage

Version `1.1` brings a redesigned API which closely follows fastText command-line interface. You can still use the old methods, 
but they are marked as obsolete: their test coverage is limited and no fixes will be issued for these methods if bugs are discovered.

### Supervised model training

The simplest use case is to train a supervised model with default parameters. You just create a `FastTextWrapper` and call `Supervised()`.

```c#
var fastText = new FastTextWrapper();
fastText.Supervised("cooking.train.txt",  "cooking", FastTextArgs.SupervisedDefaults());
```

Note the arguments:

1. We specify an input file with one labeled example per line. Here we use Stack Overflow cooking dataset from Facebook:
https://dl.fbaipublicfiles.com/fasttext/data/cooking.stackexchange.tar.gz. You can find extracted files split into training
and validation sets in `UnitTests` directory in this repository.
2. Your model will be saved to `cooking.bin` and `cooking.vec` with pretrained vectors will be placed if the same directory.
3. Here we use `FastTextArgs.SupervisedDefaults()` to get default argumets for supervised training. It's a good starting point and is
the same as calling fastText this way:

```bash
./fasttext supervised -input cooking.train.txt -output cooking
```

### Loading models

Just specify a path to the `.bin` model file:

```c#
var fastText = new FastTextWrapper();
fastText.LoadModel("model.bin");
```

### Using pretrained vectors

To use pretrained vectors for your supervised model, do this:

```c#
var fastText = new FastTextWrapper();
            
var args = FastTextArgs.SupervisedDefaults();
args.PretrainedVectors = "cooking.unsup.300.vec";
args.dim = 300;

fastText.Supervised("cooking.train.txt", "cooking", args);
```

Here we just get default training arguments, supply a path to pretrained vectors file and adjust vector dimension accordingly.

**Important!** Be sure to always check the dimension of your pretrained vectors! Many vectors on the internet have dimension `300`,
but default dimension for fastText supervised model training is `100`.

### Testing the model

Now you can easily test a supervised model against a validation set. You can specify different values for `k` and `threshlod` as well.

```c#
var result = fastText.Test("cooking.valid.txt");
```

You will get an instance of `TestResult` where you can find aggregated or per-label metrics:

```c#
Console.WriteLine($"Results:\n\tPrecision: {result.GlobalMetrics.GetPrecision()}" +
                            $"\n\tRecall: {result.GlobalMetrics.GetRecall()}" +
                            $"\n\tF1: {result.GlobalMetrics.GetF1()}");
```

You can even get a precision-recall curve (aggregated or per-label)! Here is an example of exporting an SVG plot with cross-platform
[OxyPlot library](https://oxyplot.github.io):

```c#
var result = fastText.Test("cooking.valid.txt");
var curve = result.GetPrecisionRecallCurve();

var series = new LineSeries {StrokeThickness = 1};
series.Points.AddRange(curve.Select(x => new DataPoint(x.recall, x.precision)).OrderBy(x => x.X));

var plotModel = new PlotModel
{
    Series = { series },
    Axes =
    {
        new LinearAxis {Position = AxisPosition.Bottom, Title = "Recall"},
        new LinearAxis {Position = AxisPosition.Left, Title = "Precision"}
    }
};

using (var stream = new FileStream("precision-recall.svg", FileMode.Create, FileAccess.Write))
{
    SvgExporter.Export(plotModel, stream, 600, 600, false);   
}
```

![](docs/prec-rec.png)

### Training unsupervised models

Coming soon!

For now you can train unsupervised models with a deprecated `FastTextWrapper.Train()` method.

### Automatic hyperparameter tuning

Coming soon!

### Getting logs from the wrapper

`FastTextWrapper` can produce a small amount of logs mostly conserning native library management. You can turn logging on by providing an
instance of `Microsoft.Extensions.Logging.ILoggerFactory`. In this example we use Serilog with console sink.

You can also inject your standard `IloggerFactory` through .NET Core DI.

```c#
Log.Logger = new LoggerConfiguration()
                .MinimumLevel.Debug()
                .WriteTo.Console(theme: ConsoleTheme.None)
                .CreateLogger();

var fastText = new FastTextWrapper(loggerFactory: new LoggerFactory(new[] {new SerilogLoggerProvider()}));
```

### Handling native exceptions

In version `1.1` I've added much better native error handling. Now in case of most native errors you will get a nice
`NativeLibraryException` which you can inspect for detailed error description.

## Windows Requirements

Since this wrapper uses native C++ binaries under the hood, you will need to have Visual C++ Runtime Version 140 installed when 
running under Windows. Visit the MS Downloads page (https://support.microsoft.com/en-us/help/2977003/the-latest-supported-visual-c-downloads) 
and select the appropriate redistributable. 

## FastText C-style API

If you are interested in using FastText with C-style API, here is my fork of the official library: https://github.com/olegtarasov/fastText.